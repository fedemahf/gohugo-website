---
title: Combining AWS CodeCommit and Github Actions
author: Federico Mahfoud
date: 2023-06-06
---

## Context

AWS provides a great set of tools to support both the development and the infrastructure of applications. But sadly, they don't provide pipelines for iOS. Alternatively, EC2 Mac machines can be used. However, at Ensolvers we decided to opt for a low-cost version by integrating with Github repository and Github Actions.

## Implementation
Basically our solution mirrors the content from the CodeCommit repo to Github, so we can make use of Github Actions. To accomplish this integration, we've created a Lambda function in AWS with Python 3.9 as runtime which is triggered by each push to a CodeCommit repository. This function uses the boto3 library to get a GitHub Personal Access Token stored in AWS Secrets Manager and the urllib3 library to send an HTTP POST request to the GitHub API to create a dispatch event in a GitHub Actions workflow - you can have a look to [Create a workflow dispatch event](https://docs.github.com/en/rest/actions/workflows?apiVersion=2022-11-28#create-a-workflow-dispatch-event) for more details on this. In this case, the dispatch event will trigger the workflow.

With git authentication we explored different approaches. First, we tried SSH authentication but then the implementation became too complex: it required us to create an SSH key, store the private key into a secret manager along with the SSH user, then set up SSH keys in AWS/GitHub, start up an SSH agent in the workflow, etc. For the sake of simplicity, we decided to create a new GitHub user for this project and get a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) from that user. For AWS, we've created a new [IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html) and generated [HTTPS Git credentials for AWS CodeCommit](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html).

This way, we can access all the repositories from the user through the HTTPS protocol.

```sh
# GitHub
git clone https://git:GITHUB_TOKEN@github.com/TEAM_NAME/REPO_NAME

# AWS
git clone https://AWS_USER:AWS_PASSWORD@git-codecommit.AWS_REGION.amazonaws.com/v1/repos/REPO_NAME
```

Note that the password should be URL-encoded. (That's an URL, after all.)

Now, after this consideration, the GitHub Actions workflow has two jobs:\
- Pull from CodeCommit
- Build and deploy

In the following two subsections we describe the concrete actions that need to be added to the [.github/workflows](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) file to implement the pipeline

## Pull from CodeCommit

Checkout GitHub repository.

```yml
- name: GitHub - Checkout repository
  uses: actions/checkout@v2
  with:
    ref: ${{ env.CHECKOUT_BRANCH }}
    fetch-depth: 0
```

Add a new remote with the CodeCommit repository.

```yml
- name: CodeCommit - Add remote
  run: git remote add aws ${{ env.AWS_REPOSITORY_FULL_URL }}
```

Pull all remotes and branches. It's important to have all branches, otherwise the next step can fail.

```yml
- name: Git - Pull all
  run: git pull --all
```

Pull the required branch from CodeCommit.

```yml
run: git pull --ff-only aws ${{ env.CHECKOUT_BRANCH }}
```

Push the required branch to Github repo.

```yml
- name: GitHub - Push branch
  run: git push origin ${{ env.CHECKOUT_BRANCH }}
```

## Build and deploy

Checkout GitHub repository.

```yml
- name: GitHub - Checkout repository
  uses: actions/checkout@v2
  with:
    ref: ${{ env.CHECKOUT_BRANCH }}
```

Rewrite each submodule URL to use special HTTPS credentials instead of using SSH.

```yml
- name: GitHub - Set CodeCommit submodules URL
  run: |
    git submodule set-url submodule1-name ${{ env.SUBMODULE1_FULL_URL }}
    git submodule set-url submodule2-name ${{ env.SUBMODULE2_FULL_URL }}
```

Init and update submodules.

```yml
- name: GitHub - Init and update submodules
  run: |
    git submodule init
    git submodule update
```

Finally, build code and deploy.

## Conclusion

In this post we've described a way in which we can integrate CodeCommit with Github Actions to automate iOS building, avoiding to do the same process in a non-containerized way via a reserved EC2 Mac-compatible machine.

<small>*This article was originally posted on [Ensolvers blog](https://www.ensolvers.com/post/combining-aws-codecommit-and-github-actions).*</small>