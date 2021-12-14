---
layout: post
title: An easy way to create internal documentation pages 
summary: This is GitLab. Yes, it's that simple.
featured-img:
categories: Linux Code Networking
tags: [ code, notes ]
---
This is [GitLab](https://about.gitlab.com/install/){:target="_blank"}. Yes, it's that simple.
If you have a local `DNS server` and `gitlab-ce`, then you can create a place to store documentation in, hmm..., a few minutes.

At first, let's configure `gitlab-pages` on `gitlab-ce` server:
```text
vi /etc/gitlab/gitlab.rb
```
```text
# enable GitLab Pages
pages_external_url "http://your.gitlab.domain_name/"
gitlab_pages['internal_gitlab_server'] = "http://your.gitlab.domain_name"
gitlab_pages['enable'] = true
# enable control user's membership per project/repository
gitlab_pages['access_control'] = true
```
Save and exit from `vi`.

Reconfigure `GitLab`:
```text
gitlab-ctl reconfigure
```

Now, you can login to `GitLab web interface` as `root` and make sure, that `pages` are enabled. See `Configure GitLab/Admin Area - Dashboard - Features - GitLab Pages`.

`GitLab` should have runners for `CI/CD` pipeline, that will update the documentation. If there are no runners, [install them](https://docs.gitlab.com/runner/install/){:target="_blank"}

Ok, we have `GitLab` with `CI/CD` runners and enabled `GitLab Pages`. What's next?

Now you need to add the `A` entries to your DNS server:
```text
# such a record should already be
your.gitlab.domain_name 1800 IN A    192.168.2.1
# and this is a new record
*.your.gitlab.domain_name 1800 IN A    192.168.2.1
```

Relogin to `GitLab web interface` as `user` and `Create blank project`. Btw, you can choose `Create from template` and the following pages templates are available here: `Pages/Gatsby`, `Pages/Hugo`, `Pages/Jekyll`, etc. with `Pages/`. It's very simple way to create documentation website using site template for specific platform. But, we will use [MkDocs](https://www.mkdocs.org){:target="_blank"}, very cool static site generator with awesome theme - [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/){:target="_blank"}, so, at first, we will `Create blank project` (write project name (`docs`, as instance) and choose `Internal Project`).

Clone a new repository to a local machine:
```text
git clone http://your.gitlab.domain_name/user/docs.git
```

And create any simple files with mkdocs (at first, install [mkdocs in python environment](https://squidfunk.github.io/mkdocs-material/getting-started/){:target="_blank"}):
```text
cd docs
# create new mkdocs project from docs repository
mkdocs new .
$ tree .
.
├── docs
│   └── index.md
└── mkdocs.yml
```

Let's configure `mkdocs.yml`. There are a lot of possibilities here, and confgurations are easy to find. For example, use this [mkdocs.yml](https://github.com/timeforplanb123/nornir_cli/blob/master/mkdocs.yml){:target="_blank"} configuration file.

Now, create simple pipeline for updating our pages with repository updating:
```text
vi .gitlab-ci.yml
```

And add to this file (from `mkdocs-material` [documentation](https://squidfunk.github.io/mkdocs-material/publishing-your-site/#gitlab-pages){:target="_blank"}):
```text
image: python:latest

# GitLab Shared Runners with docker tag, as instance
default:
  tags:
    - docker

pages:
  stage: deploy
  only:
    - main
    - master
  script:
    - pip install mkdocs-material
    - mkdocs build --site-dir public
  artifacts:
    paths:
      - public
```

And, if we will create web-site locally, let's create `.gitignore` file:
```text
vi .gitignore
```
Add to `.gitignore`:
```text
public
```

Save and exit from `vi`.

Now, let's tap on the classic set of commands, already:
```text
git add .
git commit -m "Initial commit"
git push
```

And, if our pipeline was completed successfully (take a look on the status of last pipeline in `CI/CD - Pipelines`. It must be `passed`. You can see the same in the repository status bage, there should be a green mark here), we can check `gitlab-pages` link - `Settings - Pages` (it has `http://user.your.gitlab.domain_name`  format) and... click it.

Next, write the documentation pages in `.md`([example](https://github.com/timeforplanb123/nornir_cli/tree/master/docs){:target="_blank"}), put it in `docs` directory, add new pages in `mkdocs.yml` file and update repository with `git add . git commit -m "Add new docs" git push` commands, again.

Of course, there is nothing new here, and, of course you need to use `TLS` (`http` is not serious. See [pages documentation](https://docs.gitlab.com/ee/administration/pages/){:target="_blank"}). It's basic `GitLab` or `GitHub` feature, and the main driver here is to store documentation in the same repository with the project and update them together. But, this is just a note about a good feature :)
