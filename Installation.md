### Docker
Install [Docker](https://www.docker.com/community-edition) for you platform.

### Git Repo
Clone the repo and launch the app:

```bash
git clone --single-branch https://github.com/subzerocloud/postgrest-starter-kit example-api
cd example-api                  # Change the current directory to the newly created one
docker-compose up -d            # Launch Docker containers
```

The API server will become available at [http://localhost:8080/rest](http://localhost:8080/rest).

Try a simple request

```bash
curl http://localhost:8080/rest/todos?select=id,todo
```

### Developer Tools
Now install the developer tools that help you in your development process
Find the [latest release](https://github.com/subzerocloud/devtools/releases/) version.<br />
Download the binary and place it in your `$PATH`<br />
Sample commands for Mac with the release v0.0.9
```bash
  wget https://github.com/subzerocloud/devtools/releases/download/v0.0.9/subzero_devtools-macos-v0.0.9.gz
  gunzip subzero_devtools-macos-v0.0.9.gz
  chmod +x subzero_devtools-macos-v0.0.9
  mv subzero_devtools-macos-v0.0.9 /usr/local/bin/
  ln -s /usr/local/bin/subzero_devtools-macos-v0.0.9 /usr/local/bin/sz
```

In the root folder of your project run the command
```
sz
```

You should see a screen like this (From now on, we'll always have this running when iterating on our project)
![DevTools](https://github.com/subzerocloud/devtools/blob/master/screenshot.png?raw=true "DevTools")

