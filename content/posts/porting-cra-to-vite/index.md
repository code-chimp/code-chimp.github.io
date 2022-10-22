---
title: "Porting Cra to Vite"
description: The pros, cons, and methodology for porting a Create React App application to Vite
date: 2022-10-21T08:11:30-05:00
draft: true

categories:
- Project Architecture

tags:
- configuration
- NodeJS
---

Issues:

- css modules
  ```json lines
  moduleNameMapper: {
    '^.+\\.module\\.(css|sass|scss)$': 'identity-obj-proxy',
  },

```
- TypeScript + http 'Request' object = CRA eject = **react-app-polyfill** + jest.config:setupFiles
- **import.meta.inv** bruhaha
