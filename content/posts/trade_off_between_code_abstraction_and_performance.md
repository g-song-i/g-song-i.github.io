+++
title="Trade-off between Code Abstraction and Performance"
date=2024-07-10

[taxonomies]
categories = ["Clean Code", "Coding Convention"]
tags = ["post"]
+++

# Table of Contents
[Before Reading…](#before-reading) <br/>
[Chaotic Situation](#chaotic-situation) <br/>
[Testing Operation: More Chaotic](#testing-operation-more-chaotic) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[But isn’t it too abstract?](#isnt-it-too-abstract) <br/>
[How to decide to abstract or not to abstract?](#how-to-decide-to-abstract-or-not-to-abstract) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Resuability](#resuability) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Complexity](#complexity) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Testability](#testability) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Scalability](#scalability) <br/>
[Results?](#results) <br/>

# Before Reading…
When I was in graduate school, I didn’t really need to code in a “well-structured” manner. What I needed was to implement code to prove that my academic theories worked. So, I had an idea of “how to code,” but it definitely differed from “how to code well.”

Now, I am working as a research engineer in a company. This role still does not require me to code “well,” but I want to ensure my work is presentable and shareable with the development department. In this post, I will share the challenges I have encountered while coding.


# Chaotic Situation

While developing a proof of concept (PoC) code, I encountered a quite chaotic situation. I wrote the code in Go, with two primary components: the **collector** and the **evaluator**. The objectives are:
    - **Collector**: To fetch cloud configurations from a Cloud Service Provider (CSP) using an Open API.
    - **Evaluator**: To assess these configurations to determine their security.
Below is a portion of my project structure:

```bash
$ tree
.
├── config.yaml
├── docs
├── main.go
├── Makefile
├── pkg
│   ├── audit
│   │   ├── firewall
│   │   │   ├── common.go
│   │   │   ├── common_test.go
│   │   │   └── types.go
│   │   ├── security_group
│   │   │   ├── common.go
│   │   │   ├── SCP_VPC_002.go
│   │   │   ├── SCP_VPC_003.go
│   │   │   ├── SCP_VPC_004.go
│   │   │   ├── SCP_VPC_005.go
│   │   │   └── types.go
│   ├── auth
│   │   └── create_request.go
│   ├── config
│   │   └── config.go
│   └── util
│       └── httpclient
│           └── httpclient.go
├── scp-cspm // binary
```

- Each directory contains a `common.go`, which defines common functions used by the evaluator, and `types.go`, which is utilized when receiving responses from the CSP. Other components mainly belong to the evaluator.

- Here is an example from `security_group/common.go`:
```go
package file_storage

import (
	"mypath/pkg/config"
	"mypath/pkg/util/httpclient"
)

func listFileStorage(projectId string) (*ListResponseFileStoragesResponse, error) {
	fullURL := config.URLsConfig.PREFIX[0] + config.URLsConfig.FILESTORAGE[0]
	return httpclient.SendRequest[ListResponseFileStoragesResponse]("GET", fullURL, config.AppConfig.AccessKey, config.AppConfig.AccessSecretKey, projectId, config.AppConfig.HeaderClientType)
}

func getFileStorageDetail(projectId, fileStorageId string) (*FileStorageDetailResponseV3, error) {
    fullURL := config.URLsConfig.PREFIX[0] + config.URLsConfig.FILESTORAGE[0] + "/" + fileStorageId
	return httpclient.SendRequest[FileStorageDetailResponseV3]("GET", fullURL, config.AppConfig.AccessKey, config.AppConfig.AccessSecretKey, projectId, config.AppConfig.HeaderClientType)
}
```

- Funtionalities of this code:
    - It calls `SendRequest` with a specified type from `types.go`.
    - It processes the response and returns it.
- Characteristics of this code:
    - It is an internal function, as indicated by the lowercase first letter of the function name.
    - It uses `config`, which may **not seem directly related to this function's primary operations**.
- Although I was initially uncomfortable using `config` in this function, as it seemed indirectly related to the function’s core duties, the fact that four evaluators are already using this function led me to continue its use.

# Testing Operation: More Chaotic
After this, I integrated the Go-Gin framework into the project to create RESTful APIs and expose API documentation to the development team. Given the increased complexity of the project structure, it became necessary to create test code for each function.
  
- Consequently, I added a new `common.go` under the firewall directory:

```go
func listFirewall(projectId string) (*ListResponseFirewallListItemResponse, error) {
	fullURL := config.URLsConfig.PREFIX[0] + config.URLsConfig.FILESTORAGE[0]
	return httpclient.SendRequest[ListResponseFirewallListItemResponse]("GET", fullURL, config.AppConfig.AccessKey, config.AppConfig.AccessSecretKey, projectId, config.AppConfig.HeaderClientType)
}
```

- However, when I tried to test it, I encountered an error because I could not initialize the `config()` function during local tests; it does not support this since `config` locates the executable path and imports `config.yaml` from that position. I considered rewriting `config` to use a relative path, but I realized that would not address the core problem.
- To resolve this, I modified the code as follows:

```go
func listFirewall(projectId string, fullURL string, accessKey string, accessSecretKey string, headerClientType string) (*ListResponseFirewallListItemResponse, error) {
	return httpclient.SendRequest[ListResponseFirewallListItemResponse]("GET", fullURL, accessKey, accessSecretKey, projectId, headerClientType)
}
```

- I also created common_test.go to include the test code:

```go
func TestListFirewall(t *testing.T) {
	projectID := ""
	fullURL := ""
	accessKey := ""
	accessSecretKey := ""
	headerClientType := "OpenApi"

	response, err := listFirewall(projectID, fullURL, accessKey, accessSecretKey, headerClientType)
	if err != nil {
		t.Errorf("Failed to list firewall: %v", err)
	}
	fmt.Println(response)
}
```

This modification successfully resolved the issue.

## Isn’t it too abstract?
Okay, now I've resolved the problem. But have I really addressed the root issue? Let me summarize the current situation:
  - Common functions, such as `listFirewall`, merely act as bridges between callers and evaluators.
  - They do not directly use `config`.
  - They are internal functions and can only be called from evaluators when needed.

This setup seems simple, but is it appropriately abstract or efficient? I find myself facing new complexities.

I tried this to test performance of two functions:
```bash
for i in {1..10}; do curl -X 'GET' 'http://172.16.0.120:10902/Firewall/test-01' -H 'accept: application/json'; sleep 1;
```

(I "planned to upload testing result as images", but I was too lazy, sorry 🤣)
- Performance:
    - With an abstraction layer: approximately 400ms overall
    - Directly sending requests: approximately 325ms overall
- Of course, the difference isn't significant, as it's just a small abstraction layer in action. Also, I can't be entirely sure about the constant responsiveness of the CSP’s server, but I wanted to run this test anyway. It was more about satisfying my curiosity and seeing the practical impact of the abstraction, rather than expecting major discrepancies. Testing like this helps me understand the overhead introduced by additional layers in our architecture, even if minimal. It’s an essential part of optimizing and ensuring that we’re making informed decisions when it comes to the balance between code maintainability and system performance.

# How to decide to abstract or not to abstract?

Here, I summarize some points, which can be discussed when you decide whether to abstract the code or not. I read articles and papers, then discuss them with GPT. 🙂 Hope it will be fun

## Resuability

Reusability is paramount in software development as it significantly reduces the time and effort required to develop and maintain code. By designing modules and components that can be reused in different parts of the application or even in different projects, developers can ensure consistency, reduce errors, and accelerate development cycles. Effective reusability promotes a DRY (Don't Repeat Yourself) approach and can lead to more efficient, cleaner code.

## Complexity

Managing complexity effectively is crucial for maintaining the readability and maintainability of the codebase. Complexity can arise from various factors, such as intricate control flow, high inter-module dependencies, or simply from trying to incorporate too many features at once. Simplifying complex systems through well-thought-out design patterns, clear documentation, and modular architecture can help in minimizing the cognitive load on developers and maintaining the system more manageable over time.

## Testability

Testability is a measure of how easily a software system can be tested. This is crucial for ensuring long-term reliability and ease of changes. Highly testable systems are modular, have clear boundaries, and minimal dependencies, which allow for more straightforward unit testing. Investing in testability leads to software that is less prone to bugs and easier to update and expand without introducing errors.

## Scalability

Scalability must be considered to accommodate growth in user demand, data volume, and computational intensity without compromising performance. Designing systems that can scale effectively involves considerations of load balancing, resource management, and the ability to upgrade or expand components without significant disruptions. Scalability ensures that the software continues to perform optimally as it grows and adapts to increased operational demands.

# Results?

Finally, I decided to abstract my code for the following reasons:

- My code is primarily for a **Proof of Concept (PoC)**, so **it does not currently need to scale significantly**. The current level of abstraction should suffice.
- However, since the development team will review my code, **clarity is crucial.** Clear, well-abstracted code facilitates easier understanding and maintenance.
- Some interface code, such as authentication, making requests, and common functions, may be reused by the development team. This **reusability** underscores the need for clarity.
- The code must be **easy to test** whenever new functions are introduced, ensuring reliability and ease of maintenance.

Through this exploration, I found the process of optimizing and abstracting code to be quite engaging. It provided a valuable opportunity to reconsider best practices in coding within the context of a real-world application.