---
title: Fixing prettier for svelte when running vscode
description: Small issue I run into every now and then that I wanted to document
slug: prettier-svelte-fix
date: 2022-11-25 00:00:00+0000
image: prettier-svelte-fix.png
categories:
  - Web
tags:
  - Svelte
  - vscode
  - Prettier
  - Js
---

While using vscode with svelte and prettier I ran into an issue where prettier would not format my svelte files.

The prettier icon at the bottom right would show a little circle with a line through it.

This happens when the .prettierrc and prettier-svelte-plugin are not installed in the root folder that vscode is opened in.

To fix this you need to install the prettier-svelte-plugin and create a .prettierrc file in the root folder of your project.

```sh
npm init # if you don't have a package.json
npm install --save-dev prettier-svelte-plugin
```

```json
{
  "plugins": ["prettier-plugin-svelte"]
}
```

Or copy the prettier config from the svelte template subdirectory your working in
