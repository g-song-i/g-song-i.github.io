+++
title="The Way to Access to Google Workspace with Google Cloud’s Service Account"
date=2024-06-21

[taxonomies]
categories = ["Sample Post"]
tags = ["post"]
+++

# Table of contents
[Overview](#overview) <br/>
[Problem Statement](#problem-statement) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Error Message](#error-message) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Method 1: Setting the Quota Project](#method-1-setting-the-quota-project) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Method 2: Impersonating as a Workspace User with a Service Account](#method-2-impersonating-as-a-workspace-user-with-a-service-account) <br/>
[REVIEW](#review) <br/>
[REF](#ref) <br/>

# Overview

In this post, I will explain some challenges I encountered while implementing a function to access Google Workspace using Google Cloud’s Service Account. **Although I'm not certain this is the correct way to access Google Workspace,** my initial goal was to utilize a single service account with privileges to manage accounts as an administrator of an organization.

Before proceeding, ensure:

- The project that includes the service account has the Google Admin SDK API activated.
- The service account you intend to use to connect to Google Workspace is enabled with Domain-wide delegation.

# Problem Statement

While developing a prototype for Cloud Identity and Entitlements Management (CIEM) solutions for Google Cloud, **I faced an issue listing workspace users with the Google Admin SDK API using a Google Cloud service account.**

Here's my initial code:

```go
package workspace

import (
	"context"
	"fmt"
	"log"

	admin "google.golang.org/api/admin/directory/v1"
	"google.golang.org/api/option"
	
	"my-project/auth"
	"my-project/config"
)

func GetWorkspaceUserList() ([]*admin.User, error) {
	ctx := context.Background()
	credentialsPath := config.ServiceAccountFilePath

	serviceInterface, err := auth.NewService(ctx, "admin", option.WithHTTPClient(config.Client(ctx)))
    if err != nil {
			return nil, err
    }
	service, ok := serviceInterface.(*admin.Service)
    if !ok {
			return nil, fmt.Errorf("failed to convert service interface")
    }

	var users []*admin.User
	req := service.Users.List().Customer("my_customer").MaxResults(500).OrderBy("email")

	err = req.Pages(ctx, func(page *admin.Users) error {
		users = append(users, page.Users...)
		return nil
	})
	if err != nil {
		log.Printf("Failed to list Workspace users: %v", err)
		return nil, err
	}

	return users, nil
}
```

This configuration might be confusing:

```go
req := service.Users.List().Customer("my_customer").MaxResults(500).OrderBy("email")
```

According to the Google Workspace’s documentation it refers to:

> The unique ID for the customer's Google Workspace account. In case of a multi-domain account, to fetch all users for a customer, use this field instead of `domain`. You can also use the `my_customer` alias to represent your account's `customerId`. The `customerId` is also returned as part of the [Users](https://developers.google.com/admin-sdk/directory/v1/reference/users) resource. You must provide either the `customer` or the `domain` parameter.
> 

## Error Message

Anyway! Here is the error I encountered:

```go
Failed to list Workspace users: googleapi: Error 403: Your application is authenticating by using local Application Default Credentials. The admin.googleapis.com API requires a quota project, which is not set by default. 

Details:
[
{
"@type": "type.googleapis.com/google.rpc.ErrorInfo",
"domain": "googleapis.com",
"metadata": {
"consumer": "projects/project-number",
"service": "admin.googleapis.com"
},
"reason": "SERVICE_DISABLED"
}
]
```

- Initially, I was confused because:
    - I was using a Service Account’s key file, which I thought did not utilize Application Default Credentials (ADC).
    - The project number listed in the error message’s “consumer” section wasn’t even part of my organization.
    - I had already activated the Admin SDK API for my project.

## Method 1: Setting the Quota Project

I attempted to configure a quota project when sending the request since the error message indicated, "The [admin.googleapis.com](http://admin.googleapis.com/) API requires a quota project, which is not set by default." Here's the revised code:

```go
func ListAllWorkspaceUsers() error {
    ctx := context.Background()

    quotaProjectID := "your-project-id"

    serviceInterface, err := auth.NewService(ctx, "admin", option.WithQuotaProject(quotaProjectID))
    if err != nil {
        log.Fatalf("Service creation failed: %v", err)
        return err
    }

    service, ok := serviceInterface.(*admin.Service)
    if !ok {
        log.Fatalf("Failed to convert service interface to admin service")
        return fmt.Errorf("Failed to convert service interface")
    }

    req := service.Users.List().Customer("my_customer").MaxResults(500).OrderBy("email")
    err = req.Pages(ctx, func(page *admin.Users) error {
        for _, user := range page.Users {
            fmt.Printf("User: %s, Email: %s\n", user.Name.FullName, user.PrimaryEmail)
        }
        return nil
    })
    if err != nil {
        log.Printf("Failed to list Workspace users: %v", err)
        return err
    }

    return nil
}
```

And…. no way! it was failed 😕 the error message here:

```go
Failed to list Workspace users: googleapi: Error 403: Not Authorized to access this resource/api, forbidden
```

## Method 2: Impersonating as a Workspace User with a Service Account

After trying various authentication methods, I discovered [a helpful post](https://www.googlecloudcommunity.com/gc/Cloud-Hub/Admin-SDK-API-Returning-Status-Code-403-Cannot-Figure-Out-Root/m-p/742710#M6830) suggesting that to use the Admin SDK API effectively, the service account must impersonate an administrator of the domain; simply enabling domain-wide delegation was insufficient.

Okay, we will try. But note that, you should ensure,

- The workspace user you attempt to impersonate is aware of this process.
- Your Service Account has the "https://www.googleapis.com/auth/admin.directory.user.readonly" permission set in the Workspace Admin Console, indicating that the workspace administrator is aware of your activities.

To implement this, configure your service account as follows:

```go
config := &jwt.Config{
        Email:      cred["client_email"].(string),
        PrivateKey: []byte(privateKey),
        TokenURL:   google.JWTTokenURL,
        Scopes:     []string{admin.AdminDirectoryUserReadonlyScope},
        Subject:    config.AdminEmail,
    }
```

In my code, config library works getting global configuration for the code. So it gets admin email, which you impersonate. Otherwise, cred and privateKey are from the Service Account’s key fie. Other process are same to before. 

And… IT FINALLY SUCCEEDED! 😂

# REVIEW

I believe this method succeeded because it authenticated correctly with the necessary permissions and within the proper administrative context. Although the service account was granted permissions through its client ID, Google Workspace is designed for organizations and their human employees, necessitating this specific authentication procedure. Despite the challenges, I found the experience both stressful and enjoyable!

# REF

1. https://www.googlecloudcommunity.com/gc/Cloud-Hub/Admin-SDK-API-Returning-Status-Code-403-Cannot-Figure-Out-Root/m-p/742710#M6830
2. https://developers.google.com/admin-sdk/directory/reference/rest/v1/users/list#query-parameters