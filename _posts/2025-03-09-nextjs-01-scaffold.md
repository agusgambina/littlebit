---
layout: post
title:  "NextJS Scaffold"
date:   2025-03-09 23:00:00 -0300
categories: nextjs
comments: true
---

# Initialize the project

To create a NextJS app, step on the parent folder where your project will reside and execute

```
$ npx create-next-app@latest
```

The output will be

```
create-next-app@15.2.1
Ok to proceed? (y)
```

Answer `y`, and the following questionnaire will appear. Answer as you wish

```
✔ What is your project named? … your-project-name
✔ Would you like to use TypeScript? … No / Yes
✔ Would you like to use ESLint? … No / Yes
✔ Would you like to use Tailwind CSS? … No / Yes
✔ Would you like your code inside a `src/` directory? … No / Yes
✔ Would you like to use App Router? (recommended) … No / Yes
✔ Would you like to use Turbopack for `next dev`? … No / Yes
✔ Would you like to customize the import alias (`@/*` by default)? … No / Yes
```

A new folder with the `your-project-name` will be created.

# After initializing the project

After creating the app I do the following

## Create a gitignore

I supplement my project's `.gitignore` file with standard exclusions to prevent unnecessary files from being tracked. To do this, I utilize the `.gitignore` generator at https://www.toptal.com/developers/gitignore/. I search for common operating system and environment-specific entries like Windows, Linux, OSX, and dotenv, and then append the generated rules to my project's .gitignore file.

## Add Prettier

The first thing to do is to install Prettier as a dev dependency

```
$ npm install --save-dev prettier
```

Then create the `.prettierrc` file on the root

```
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2
} 
```

Create a `.prettierignore` file to specify which files should be ignored

```
node_modules
build
dist
.next
```

Add Prettier scripts to `package.json`

```
{
  // ... existing code ...
  "scripts": {
    // ... existing scripts ...
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
  // ... existing code ...
}
```

To add ESLint Prettier integration, execute

```
$ npm install --save-dev eslint-config-prettier
```

Then update your `.eslintrc` to include Prettier

```
{
  "extends": [
    "prettier"
  ]
} 
```
## Add the NodeJS version

To ensure consistent NodeJS versions across my projects, I always include an `.nvmrc` file. This file specifies the exact version required, simplifying development and collaboration.

```
# .nvmrc
v22.14.0
```

Implementing this provides dual benefits: it functions as real-time documentation for the project's requirement, and it automates the NodeJS version selection via a bash script when changing directories.
