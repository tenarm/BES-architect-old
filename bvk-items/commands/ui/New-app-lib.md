## Create a new application

```bash
npx nx generate @nx/react:app apps/shell --bundler=vite --style=css --e2eTestRunner=none --unitTestRunner=none --tags "scope:shell"
```

## Create a new library

```bash
npx nx generate @nx/react:lib libs/shared-ui --bundler=vite --style=css --unitTestRunner=none --tags "scope:shared-ui" --tags "scope:shared-ui"
```

```bash
npx nx generate @nx/react:lib libs/finance --bundler=vite --style=css --unitTestRunner=none --tags "scope:finance" --tags "scope:finance"
```


## Remove app or libray

```bash
npx nx generate remove shell 
```