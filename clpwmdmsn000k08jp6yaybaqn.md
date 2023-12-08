---
title: "How to handle multiple git based systems on the same Mac(hine)"
seoTitle: "How to handle multiple Git access on the same Mac"
seoDescription: "This article tells you how to configure your Mac to interact with different Git based Version Control Systems without typing any credential"
datePublished: Fri Dec 08 2023 12:45:09 GMT+0000 (Coordinated Universal Time)
cuid: clpwmdmsn000k08jp6yaybaqn
slug: how-to-handle-multiple-git-based-systems-on-the-same-machine
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/2JIvboGLeho/upload/ea7f98a49781c1d89bc0797ac3d49c69.jpeg
tags: mac, github, version-control, gitlab, aws-sso, codecommit

---

Hello everyone 👋 ! This is my first article, and my first tech blog actually.

In this article, I'll show you how I configured my Mac to work on repos hosted on my personal Github, on my company's Github, on Gitlab, and on AWS CodeCommit with AWS SSO integration.

Not rocket science 🚀, but something I've struggled with a bit and can be achieved in a number of ways.

Let's see how I did it.

## Why do I need many version control systems?

Here's my context: I work for a company as a cloud architect and need to access my company's repositories hosted by Github Enterprise with a corporate user.

My company also has an AWS organisation, and we use AWS SSO federated with Corporate ADFS to access the organisation's accounts, and we have some repositories hosted on CodeCommit.

We also have some repositories on an old Gitlab installation that is somewhere in the basement of our office.

And finally, I have my personal repositories hosted on Github under my good old username.

I assume I'm not alone in the world with this:

* Org's Enterprise Github, accessed with XYZ user
    
* Org's Gitlab, accessed with TYU user
    
* Org's AWS CodeCommit, accessed with QWE (federated) user
    
* Your personal Github, accessed with ZXC user
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701966013128/1ed1a988-093d-4a65-b673-1b1af4913864.png align="center")

![]( align="center")

I'm used to working with only one Mac, both for professional and personal projects. Therefore I need to pull and push code from/to different version control systems, in my case all Git-based, with different users and of course without being asked for credentials with every command.

I already had a lot of my Company's repositories downloaded to my Mac when I added all the other repositories, and I didn't want to reconfigure all my local Git repositories.

Does this sound familiar? If so, go ahead...

## OSX Key Chain, dear friend...

Okay, now what?

So the OSX keychain can store your credentials and you can configure your Git client to retrieve them, right? Wrong!

Of course you can, but the OSX keychain stores credentials by hostname, which means it can store your Github company credentials OR your personal credentials, because for both the host is [github.com](http://github.com).

With Key Chain you can store ONE credential per host, or at least I haven't found a way to store more than one.

So if you configure Git to use OSX Key Chain as a credential helper and you store the credentials of your personal Github user, everything will work fine when interacting with your personal repos.

But if you try to interact with your organisation's repositories, you'll get a 403.

## HTTPS vs SSH vs GRC

The three Version Control Systems supports different protocols support

| VCS | HTTPS | SSH | GRC |
| --- | --- | --- | --- |
| Github | ✅ | ✅ | ❌ |
| AWS CodeCommit | ✅ | ✅ | ✅ |
| GitLab | ✅ | ✅ | ❌ |

Since I had already configured many Github Enterprise repositories to use HTTPS, I considered my Enterprise Github to be the default.

### Enteprise Github via HTTPS

I set up git to use OSX Key Chain as credential helper in my git ***global*** config, as follow:

```bash
[credential "https://github.com"]
    helper = osxkeychain
```

and i use https connection when i clone repositories from there.

To edit your git global configuration, use the following command:

```bash
git config --global --edit
```

This snippet shows local git configuration for HTTPS connection:

```bash
[remote "origin"]
        url = https://github.com/your-org/your-repo.git
        fetch = +refs/heads/*:refs/remotes/origin/*
```

For my enterprise Github, that's enough: It uses the credentials stored in my Credentials Helper.

This way, every time I clone a repo over HTTPS and don't specify a local Git configuration, the global configuration and the OSX keychain are used for the credentials.

### Personal Github via SSH

Then, for my personal Github, i set up SSH connection.

You can do the same following [this guide](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account).

You need to tell git to use that key, and here is my ***local*** git config

```bash
[remote "origin"]
        url = git@your-shh-key-alias:your-user/your-repo.git
        fetch = +refs/heads/*:refs/remotes/origin/*
```

to edit your local git config, run the following command inside your repository's root folder

```bash
git config --edit
```

Look the url part of this configuration:

> url = [git@](mailto:git@github.com)*your-ssh-key-alias*:*your-user*/*your-repo*.git

You can see that I used an alias for my ssh key.

To do that, you need to edit your *.ssh/config* file.

Here is my *.ssh/config*

```bash
Host github.com-personal
   HostName github.com
   User git
   IdentityFile ~/.ssh/github/id_rsa_personal
   TCPKeepAlive yes
   IdentitiesOnly yes
```

The *Host* parameter is your ssh key alias, so my url looks like

> url = [git@](mailto:git@github.com)**github.com-personal:***your-user*/*your-repo*.git

The *IdentityFile* is the path to your key file on disk, so it depends on where you have saved it.

Pay attention to the *user* parameter: this is not your Git user, but the user for the ssh connection.

This way, every time I clone one of my personal repositories, I have to edit my local Git configuration.

In my case, I have a lot more new repositories from my company than personal repositories and therefore I decided to keep HTTPS as default for my company repositories: It requires less configuration.

You know, programmers are lazy 🦥...

## Gitlab via SSH

I used ssh for connecting to Gitlab too, [here Gitlab doc to ssh connection](https://docs.gitlab.com/ee/user/ssh.html).

The local configuration is exactly the same as for Github. So you need another alias for your ssh key and have to set up your local Git repository to use your alias.

## CodeCommit via GRC

As i said, our AWS Accounts are part of our AWS Organisation and we access them with AWS SSO federated with our ADFS.

Googling around i found [this documentation page from AWS](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-git-remote-codecommit.html) that starts with:

> If you want to connect to CodeCommit using a root account, federated access, or temporary credentials, you should set up access using **git-remote-codecommit**.

Bingo! 🎯

Here how it works:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702034337823/e4e95b87-8956-4146-a197-33116ff5a7c4.png align="center")

![]( align="center")

First you have to install ***git-remote-codecommit* python package.**

[This documentation page tells you how](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-git-remote-codecommit.html).

Then you need to create a local AWS profile tied to the account/region where your repositories are stored and configure AWS SSO.

Just follow [this guide from AWS](https://docs.aws.amazon.com/cli/latest/userguide/sso-configure-profile-token.html) to do so.

Finally, you have to clone your repository using that package.

Here's mine local git repository configuration:

```bash
[remote "origin"]
        url = codecommit://your-aws-profile@your-repo
        fetch = +refs/heads/*:refs/remotes/origin/*
```

let's take a look to the *url* parameter:

> url = codecommit://your-aws-profile@your-repo

*codecommit* is the protocol, it tells git to use our python package.

*your-aws-profile* refers to your AWS Profile name.

This way, Git commands are executed on this repository with the Python package and with my AWS profile.

If you have a repository in different accounts, you need to set up different profiles and use them accordingly.

## Wrap up

Here is how it looks like at the end of the story:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702037941106/7ddad4a6-a484-49ec-b910-34d396326741.png align="center")

![]( align="center")

With the above configuration, I can easily switch between local folders and use Git without being asked for any credentials and without 403. 🎉

I tried to keep the configuration overhead low for the most commonly used version control system (Enterprise Github) and I used GRC for CodeCommit because I don't want to specify a profile when I run my Git command.

i have aliases and want to use them the same way regardless of the account, and this configuration hides the profile specification from the commands.

But if I didn't already have many cloned repositories with HTTPS, I would use SSH for enterprise Github as well and remove the OSX key chain from my configuration.

I hope this was helpful, thanks for reading!

👋👋