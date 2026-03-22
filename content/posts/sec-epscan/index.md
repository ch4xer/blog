---
title: (论文&代码阅读)EPScan:Automated Detection of Excessive RBAC Permissions in Kubernetes Applications.
author: ch4xer
date: 2025-07-31T16:20:47+08:00
categories:
  - 安全研究
draft: true
---

# EPScan:Automated Detection of Excessive RBAC Permissions in Kubernetes Applications.

> 遥想去年开题思考博士研究点的时候，就有分析Kubernetes RBAC过度授权的问题，那个时候刚好就看到了这篇论文，只好含恨想别的。今年复旦的团队总算把源码放出来了，于是就有了这篇阅读笔记。

[Paper](https://ieeexplore.ieee.org/abstract/document/11023379/)
[Source Code](https://github.com/seclab-fudan/EPScan)

## Background

RBAC is a critical component of Kubernetes security, but excessive permissions can lead to vulnerabilities. Currently, there is no work that identify whether the application requires all the permissions granted by the developer or, some of them are excessivly granted. This work aims to automate the detection of excessive RBAC permissions in Kubernetes applications, which can help developers and security teams to identify and mitigate potential security risks.

## Approach

Given a third-party application, EPScan (the tool developed by the authors) determines the permissions required by application pods based on source code analysis and compares them with the permissions granted by the RBAC policies to identify excessive permissions.

Challenges:

- (C1) How to identify source code entry for executables running in the pod, as cloud native applications always consist of multiple pods?
- (C2) How to identify and model resource access behavior in the source code?
- (C3) How to determine the reachable resource access for each program?

![Arichitecture](./architecture.png)

### RBAC-centered Configuration Analysis

This step extracts and formats the RBAC permissions from configuration files.

### LLM-assisted Pod-Program Matching (to solve C1)

1. download the image of the application pod
2. generates a prompt based on the contents of the startup commands and scripts collected from the container image
3. use LLM to identify the executable files in the image (use few-shot learning to format the output from LLM), and extracts them from it.
4. locate the code entry of these executables in source code repository via runtime information within the executable files, such as **the file path of the function's source code**, then identify the `main` function in the code entry.

### RefGraph-based Program Behavior Analysis

After finding the source code entry, it's time to dig the program behaviors where RBAC permissions involved.

1. locates the call sites for all resource access APIs
2. constructs the RefGraph of the application and determine the reachability of call sites.

#### Call sites locating

Kubernetes provides two libraries for resource access: client-go and controller-runtime

- client-go: most common used library for resource access.
- controller-runtime: library for building K8s controllers and operators.

This paper analyse the api usages of these two libraries, and summarize the following types of call sites:

1. the resource is the receiver of api call, EPScan captures the type of the receiver to represent the resource type.

   ```go
   dep := clientset.AppsV1().Deployments(ns)
   dep.Delete(ctx, "demo-deployment", opts)
   ```

2. the resource is the parameter of api call, EPScan locates the first place on the data flow where the object exists as a non-common interface type and treats that type as the resource type.

   ```go
   obj := &appsv1.Deployment{
       ...
   }
   client.Delete(ctx, obj, opts)
   ```

3. the resource is collected by watching from other places , and filterd with certain conditions. EPScan models these APIs by performing data flow analysis on the parameters to verify if the input parameters are constant strings, collecting these constant strings to represent the resource type.

   ```go
   lw := cache.NewListWatchFromClient(client, "deployments", ns, field)
   ```

#### RefGraph Construction

After identifying the resource access behavior, an accurate reachability analysis is required to identify the resource access reachable from each main entry. However, the traditional call graph construction method is ineffective on third-party applications, cause

- most function call chains are very deep, containing a large number of external library functions, whch affect the accuracy of traditional call-graph analysis.
- Second, third-party applications sometimes use callback and function pointers, whch are not well supported by call graph analysis.

So, The differences between Call Graph and RefGraph?

A Call graph is a straightforward concept:

- Nodes: Represent individual functions or methods
- Edges: Represent a direct call from one function to another.

A Reference Graph is created by the author, **which extends the scope of tracking**. Each edge of RefGraph represents one of the following types of relationship between function, global variable of type.

The RefGraph is constructed using CodeQL.

##### Function call

function A contains a direct call to function B

```
predicate cgEdge(Function a, Function b) { b.getACall().getRoot().(FuncDecl).getFunction() = a }
```

##### Function reference in another function

function A contains a reference of function B

> **Reference is different from being called**. A reference means its name is used, for example, to assign it to a variable or pass it as an argument.

```
predicate methodRefEdge(Function a, Function b) { b.getAReference().getParent+().(FuncDecl).getFunction() = a and a != b}
```

Example:

```go
func workerTask() {
}

func setupAndRun() {
    // Here, 'workerTask' is referenced and assigned to a variable.
    // It is NOT directly called as workerTask().
    taskToRun := workerTask

    // The task is executed later via the variable.
    taskToRun()
}
```

##### Package import

A package containing `init` function is imported in main package, which triggers `init` automatically.

```
class EntryFunction extends Function {
  EntryFunction() { this.hasQualifiedName(_, "main") }
}

class InitFunction extends Function {
  InitFunction() { this.getName() = "init" }

  Package getActualPackage() {
    exists(PackagedFile pkgFile
      | this.getFuncDecl().getFile() = pkgFile
      | result = pkgFile.getPackage())
  }
}

predicate entryCallInit(Function a, Function b) {
    initCalling(a, b)
}

predicate initCalling(EntryFunction entry, InitFunction init) {
  packageImports*(entry.getPackage(), init.getActualPackage())
}
```

##### Global variable usage in function

```
predicate functionUseGv(Function f, GlobalVar gv) {
  f.getBody() = gv.getAReference().getParent+()
}
```

##### Type reference in function

A struct type is refered in function.

```
predicate typeRefInFunction(Function f, Type typ) {
  isNamedStructType(typ) and
  typ.getEntity().getAReference().getParent*() = f.getBody()
}
```

Example:

```
type Pod struct {
  ApiVersion string
  Kind       string
}

// This function references the 'Pod' type.
func createPodObject() Pod {
  // A reference to the 'Pod' type is used here to declare the variable 'p'.
  p := Pod{ApiVersion: "v1", Kind: "Pod"}
  return p
}
```

##### Global variable(gv) declaration contains another gv

```
class GlobalVar extends ValueEntity {
    GlobalVar() {
        this.getDeclaration().getParent().getParent().getParent() instanceof File
    }

    GlobalVarAssign getDeclAssign() {
        result = this.getDeclaration().getParent()
    }
}

predicate gvDefByGv(GlobalVar gv, GlobalVar gvUsed) { gv.getDeclAssign().getAUsedVar() = gvUsed }
```

##### Global variable declaration contains function

```
predicate gvDefByFunction(GlobalVar gv, Function f) {
  gv.getDeclAssign() = f.getAReference().getParent+().(GlobalVarAssign)
}
```

##### Call method on receiver of specific type

```
predicate methodToReceiverType(Type typ, Function f) {
  f.getFuncDecl()
    .getAChild()
    .(ReceiverDecl)
    .getAChild*()
    .(TypeName)
    .getType() = typ
}
```



#### Reachability Approximation

After RefGraph construction, EPscan can utilize the RefGraph to approximate the reachability **from resource access API call sites to code entry points**, thus filtering out unreachable API call sites.



```
predicate edges(RefGraph::PathNode pred, RefGraph::PathNode succ) {
    RefGraph::edges(pred, succ)
}


class EntryFunction extends Function {
  EntryFunction() { this.hasQualifiedName(_, "main") }
}


from
    RefGraph::PathNode start,
    RefGraph::PathNode end,
    EntryFunction entryPoint,
    K8sApiCall apiCall
where
    edges+(start, end)
    and start.asFunction() = entryPoint
    and end.asFunction() = getParentFunction(apiCall.getCall())
select
    // Entrypoint, Callsite Parent, Resource Type, Verb, Callsite Loc
    entryPoint.getDeclaration().getLocation().getFile(),
    end.asFunction().getQualifiedName(),
    apiCall.getResourceName(),
    apiCall.getVerbName(),
    apiCall.getLocation().toString()

```



### Excessive permission detection

## Evaluation

## Code

````

```

```
