---
published: false
title: Golang - How I learned Go!!
cover_image: 'https://github.com/kodelint/blog-images/raw/main/common/01-learn-go.png'
description: null
tags: 'golang, programming'
series: null
canonical_url: null
---

![](https://github.com/kodelint/blog-images/raw/main/common/02-learn-go.png)

As **Operations Engineer** I was always a scripting guy, however as I transitioned and adopting **DevOps Culture**. I started spending more time learning _**programming languages**_.

My _**obsession**_ with Go started with my `terraform` journey. I was fascinated with the fact that `Go` produces a single binary for the platform and can run without any external dependency, unlike `python`. This was more than enough to get me started on this.....

> **The best way to learn a programming language is to write code in it**

I had this requirement where I wanted to sync external _**GitHub Repositories**_ to our internal _**GitHub Repositories**_. So, _**I though let me write the code in `Go`**_

### Requirements:

 1. **Read the YAML:** The program should read a `YAML` file, which is organized something like below:

```yaml
RepositoryList:
  - internal_org: "internal-modules"
    internal_repo_name: "internal-aws-autoscaling-module"
    internal_repo_description: "This module helps to create autoscaling groups and corresponding launch configurations for AWS"
    github_repo_name: "terraform-aws-modules/terraform-aws-autoscaling"
    repo_tags:
      - "v3.6.0"
      - "v4.4.0"
      - "v4.5.0"
```

So the `YAML` file is organized to provide a **List of Repositories**. I used [**yaml.v2**](http://"gopkg.in/yaml.v2") module, which provides the ability to **encode** and **decode** `YAML` values.

I would also need a _**data structure**_ to hold the values for me. So I used struct to define a type and variable to store the data in it. Something like below

```golang
type GithubRepo struct {
 InternalOrg string `yaml:"internal_org"`
 InternalRepoName string `yaml:"internal_repo_name"`
 InternalRepoDescription string `yaml:"internal_repo_description"`
 GitHubRepoName string `yaml:"github_repo_name""`
 RepoTags []string `yaml:"repo_tags"`
}

type RepositoryList struct {
 ListOfRepositories []GithubRepo `yaml:"RepositoryList"`
}
```

Let me talk a little bit about the `yaml:”internal_org”` these are called `Tags`. `Tags` are a way to attach additional information to a `struct` field. `Tags` use the `key:value` format. It’s not a strict rule, but a convention, which provides built-in parsing.

Different packages use **`tags`** for different reasons. We can use them used them in encoding libs, like `json`, `xml`, `bson`, `yaml` etc

Now all I need is to read the `YAML` file and store the data in `RepositoryList` type variable

```golang
var yamlFile RepositoryList

f, err := os.Open("Repositories.yml")
 if err != nil {
  log.Fatalf("os.Open() failed with '%s'\n", err)
 }
 defer func(f *os.File) {
  err := f.Close()
  if err != nil {

}
 }(f)

newDecoder := yaml.NewDecoder(f)
err = newDecoder.Decode(&yamlFile)
if err != nil {
  log.Fatalf("newDecoder.Decode() failed with '%s'\n", err)
}
```

> **Quick Tip**: never forget to **close** the file after reading it. However you might need it again. To avoid opening and closing multiple time you can use the defer key word and an anonymous function to check for closing errors

**2. Fetch Credentials from Vault**: All `credentials` are in vault the program needs to fetch them from `vault` based on couple of `environment` variables like

* `VAULT_ADDRESS` , `VAULT_APP_ROLE_ID` , `VAULT_APP_ROLE_SECRET_ID` , `VAULT_SECRET_PATH`

I used [“hashicorp/vault/api”](http://"github.com/hashicorp/vault/api") module for the same.

**3. Connect to Internal and External GitHub APIs**: In order to fetch the external repository and push it internal one I needed to make connection to both external and internal. I used “[**github**](http://"github.com/google/go-github/v38/github")” module accomplish that.

**4. Do something with the Data**: Now I have the data stored in `yamlFile` variable, I have the `GitToken` from `Vault` . It’s time to access the data individually and do something with it. Data can be accessed something like this: Because it is **`List`** so I can iterate over it and hold the chunk of data in temporary loop variable `v`

```golang
for _, v := range yamlFile.RepositoryList {
  // Now I can access the content individually
  // and do something about it
  fmt.Println(v.InternalOrg)
}
```

All the data read from `YAML` file can now access under `v.<<FieldName>>`. Something like this

`v.InternalOrg` , `v.InternalRepoName` , `v.GitHubRepoName` , `v.RepoTags` etc

![](https://github.com/kodelint/blog-images/raw/main/common/01-learn-go.png)

### Learnings

So with this exercise was I was able to quite few things, here are some of them:
>
* **Go `Structs`**
* **Go `Structs` Tags**
* **Go `DataTypes`**
* **How to use external modules**
* **Reading Files**
* **Encoding and Decoding `YAML` Data**
* **Access data with `FieldName`**

Learning a new programming language is always fun and when you choose something to build along side then it becomes more involved and interesting!

## Happy Coding!!
