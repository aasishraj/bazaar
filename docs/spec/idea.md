The project idea is a platform that lets you sell / share your ai ide setup.

For simplicity, we will focus on having this feature only for codex cli by openai.

the directory `project-root/docs/codex-cli/` contains the documentation of all files, config that can be used to setup the codex ide.
hereafter, the mention of config includes all the config files including agents skills, etc.
note: this platform is only to share global configs.

in the end there would be 2 components to this project:
1. a next js web page that authenticates with google oauth, the marketplace that enables you to list your config.
    - the config wouldn't be shown there. rather a config code would be displayed along with a description, image, etc.
    - the config code is auto generated.

2. a python cli tool that enables the packaging and sharing of these config files.
    - `bazaar login`: opens up the above webpage from where the user needs to authenticate using oauth and writes the credentials to `~/.bazaar/credentials.json` file
    - `bazaar init`: checks if the user is authenticated, if not, authenticate the user, then create the directory `~/.bazaar/<ide>/config/`. the ide can now be given as `codex` since we are keeping it to a minimum.
    - `bazaar logout`: clears the `credentials.json`
    - `bazaar setup #id`: if the user has access to that specific config id, it basically pulls the config from the web page's api and stores it inside `~/.bazaar/<ide>/config/#id/` and then places those files in the appropriate location for global config. if the user has no access yet, the cli asks a question whether to request for access.
    - `bazaar upload --name <name> --description <description> --visibility <private(default)/public>`: packages your whole global config unless it is from an existing global config. if it is a modification of an existing config, that would be the parent.

It is to be a git based system.