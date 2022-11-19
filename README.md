<img src="logo.png" width="300">

# Staticman [![coverage](https://img.shields.io/badge/coverage-81%25-yellow.svg?style=flat)](https://github.com/eduardoboucas/staticman) [![Build Status](https://travis-ci.org/eduardoboucas/staticman.svg?branch=master)](https://travis-ci.org/eduardoboucas/staticman) [![JavaScript Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://standardjs.com)

> Static sites with superpowers

## Info for Dan!!!

I forked this repo from [the official Staticman repo](https://github.com/eduardoboucas/staticman) in order to deploy my own instance to Azure.
Previously I had my own instance deployed to Heroku, which you can [view the archived code for here](https://github.com/deadlydog/deadlydog.github.io-staticman-heroku), but when Heroku removed their free tier I moved to Azure.

To deploy to Azure, in the Azure portal I created a new Web App and granted it permissions to access this git repo.
It created [the GitHub action](.github/workflows/master_dansblog-staticman.yml) that is used to deploy the code to the Web App.
Rather than enabling CI/CD, you must manually trigger [the GitHub action](.github/workflows/master_dansblog-staticman.yml) to deploy the code to Azure.

In the Azure portal for the Web App, I added the following Application Settings:

- `AKISMET_API_KEY`: [redacted]
- `AKISMET_SITE`: https://blog.danskingdom.com
- `GITHUB_TOKEN`: [redacted]
- `NODE_ENV`: production
- `RSA_PRIVATE_KEY`: [redacted]
- `SCM_DO_BUILD_DURING_DEPLOYMENT`: true

### Initial problem after deploying to Azure and resolution

After deploying the code to Azure, I was getting a `You do not have permission to view this directory or page.` error message when viewing the site root <https://dansblog-staticman.azurewebsites.net>.
[This StackOverflow answer](https://stackoverflow.com/a/69649561/602585) pointed me in the direction of the problem being there was no root web.config file in the site.
[This Azure Docs issue](https://github.com/MicrosoftDocs/azure-docs/issues/11848) pointed me to [the official MS docs that described the problem](https://learn.microsoft.com/en-us/azure/app-service/quickstart-nodejs?tabs=windows&pivots=development-environment-vscode#configure-the-app-service-app-and-deploy-code) and mention the solution is to add the `SCM_DO_BUILD_DURING_DEPLOYMENT` application setting to have Azure automatically create the web.config file when the site is deployed.
Afterward, I found this same solution mentioned in [this StackOverflow answer](https://stackoverflow.com/a/70721732/602585).

Once the web.config was created, I ran into a new problem where the page would load for about a minute and eventaully give a 500 status code message saying `dansblog-staticman.azurewebsites.net is currently unable to handle this request.`.
Enabling diagnostic logs on my Web App in the Azure Portal showed the following error:

> IIS was not able to access the web.config file for the Web site or application.
> This can occur if the NTFS permissions are set incorrectly.
> IIS was not able to process configuration for the Web site or application.
> The authenticated user does not have permission to use this DLL.

This error message led me to [this StackOverflow answer](https://stackoverflow.com/a/17087978/602585) showing that the problem was my node app was listening to the wrong port.
For Azure node apps, they need to obtain the port number to listen to from the `process.env.PORT` variable.
This required a small change to the app, which you can read about in the section below.

### Things I changed

Below is the list of things I changed from the original Staticman repo to make it work on Azure:

- In [PR #2](https://github.com/deadlydog/deadlydog.github.io-staticman/pull/2) I updated the [server.js](server.js) file to use the `process.env.PORT` variable for the port number, rather than the PORT environment variable from the config.

In addition, I also updated the GitHub Action used for deployments:

- In [PR #1](https://github.com/deadlydog/deadlydog.github.io-staticman/pull/1) I changed it to zip and unzip the artifacts being stored, reducing deployment time from 40+ minutes to about 6 minutes.

## Introduction

Staticman is a Node.js application that receives user-generated content and uploads it as data files to a GitHub and/or GitLab repository. In practice, this allows you to have dynamic content (e.g. blog post comments) as part of a fully static website, as long as your site automatically deploys on every push to GitHub and/or GitLab, as seen on [GitHub Pages](https://pages.github.com/), [GitLab Pages](https://about.gitlab.com/product/pages/), [Netlify](http://netlify.com/) and others.

It consists of a small web service that handles the `POST` requests from your forms, runs various forms of validation and manipulation defined by you and finally pushes them to your repository as data files. You can choose to enable moderation, which means files will be pushed to a separate branch and a pull request will be created for your approval, or disable it completely, meaning that files will be pushed to the main branch automatically.

You can download and run the Staticman API on your own infrastructure. The easiest way to get a personal Staticman API instance up and running is to use the free tier of Heroku. If deploying to Heroku you can simply click the button below and enter your config variables directly into Heroku as environment variables.

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy)

## Requirements

- Node.js 8.11.3+
- npm
- A [personal access token](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/) for the GitHub and/or GitLab account you want to run Staticman with
- An RSA key in PEM format

## Setting up the server on your own infrastructure
NOTE: The below steps are not required if deploying to Heroku. To deploy to Heroku, click the above deploy button and enter your configuration variables in the Heroku Dashboard.

- Clone the repository and install the dependencies via npm.

  ```
  git clone git@github.com:eduardoboucas/staticman.git
  cd staticman
  npm install
  ```

- Create a development config file from the sample file.

  ```
  cp config.sample.json config.development.json
  ```

- Edit the newly-created config file with your GitHub and/or GitLab access token, SSH private key and the port to run the server. Click [here](https://staticman.net/docs/api) for the list of available configuration parameters.

- Start the server.

  ```
  npm start
  ```

Each environment, determined by the `NODE_ENV` environment variable, requires its own configuration file. When you're ready to push your Staticman API live, create a `config.production.json` file before deploying.

Check [this guide](docs/docker.md) if you're using Docker.

## Setting up a repository

Staticman runs as a bot using a GitHub and/or GitLab account, as opposed to accessing your account using the traditional OAuth flow. This means that you can give it access to just the repositories you're planning on using it on, instead of exposing all your repositories.

To add Staticman to a repository, you need to add the bot as a collaborator with write access to the repository and ask the bot to accept the invite by firing a `GET` request to this URL:

```
http://your-staticman-url/v2/connect/GITHUB-USERNAME/GITHUB-REPOSITORY
```

## Site configuration

Staticman will look for a config file. For the deprecated `v1` endpoints, this is a  `_config.yml` with a `staticman` property inside; for `v2` endpoints, Staticman looks for a `staticman.yml` file at the root of the repository.

For a list of available configuration parameters, please refer to the [documentation page](https://staticman.net/docs/configuration).

## Development

Would you like to contribute to Staticman? That's great! Here's how:

1. Read the [contributing guidelines](CONTRIBUTING.md)
1. Pull the repository and start hacking
1. Make sure tests are passing by running `npm test`
1. Send a pull request and celebrate

## Useful links

- [Detailed Site and API Setup Guide](https://travisdowns.github.io/blog/2020/02/05/now-with-comments.html)
- [Improving Static Comments with Jekyll & Staticman](https://mademistakes.com/articles/improving-jekyll-static-comments/)
- [Hugo + Staticman: Nested Replies and E-mail Notifications](https://networkhobo.com/2017/12/30/hugo-staticman-nested-replies-and-e-mail-notifications/)
- [Guide on How to Setup Staticman with Gatsby](https://github.com/jovil/gatsby-staticman-example)
