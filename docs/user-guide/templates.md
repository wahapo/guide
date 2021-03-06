---
layout: main
title: Templates
category: User Guide
menu: menu
toc:
    - title: Templates
      url: "#templates"
    - title: Finding templates
      url: "#finding-templates"
    - title: Using a template
      url: "#using-a-template"
    - title: Creating a template
      url: "#creating-a-template"
    - title: Removing a template
      url: "#removing-a-template"
---
# Templates

Templates are snippets of predefined code that people can use to replace a job definition in a [screwdriver.yaml](./configuration). A template contains a series of predefined steps along with a selected Docker image.

## Finding templates

To figure out which templates already exist, you can make a `GET` call to the `/templates` [API](./api) endpoint. You can also see templates in the UI at `<YOUR_UI_URL>/templates`.

Example templates page:
![Templates](assets/templates.png)

## Using a template

To use a template, define a `screwdriver.yaml` with a `template` key. In this example, we are using the [nodejs/test template](https://cd.screwdriver.cd/templates/nodejs/test).

Example `screwdriver.yaml`:

```yaml
jobs:
    main:
        requires: [~pr, ~commit]
        template: nodejs/test@1.0.3
```

You can also refer to a template version with a tag name if the template has one. If no version is specified, the latest published version will be used.

Example:
```yaml
jobs:
    main:
        requires: [~pr, ~commit]
        template: nodejs/test@stable
```


Screwdriver takes the template configuration and plugs it in, so that the `screwdriver.yaml` becomes:

```yaml
jobs:
    main:
        image: node:8
        requires: [~pr, ~commit]
        steps:
          - install: npm install
          - test: npm test
        environment:
            FOO: bar
        secrets:
              - NPM_TOKEN
```

### Wrap
Wrapping is when you add commands to run before and/or after an existing step. To wrap a step from a template, add a `pre` or `post` in front of the step name.

Example:
```yaml
jobs:
    main:
        requires: [~pr, ~commit]
        template: nodejs/test@1.0.3
        steps:
            - preinstall: echo pre-install
            - postinstall: echo post-install
```

This will run the command `echo pre-install` before the template's `install` step, and `echo post-install` after the template's `install` step.

### Replace
To replace a step from a template, add your command with the same template's step name.

Example:
```yaml
jobs:
    main:
        requires: [~pr, ~commit]
        template: nodejs/test@1.0.3
        steps:
            - install: echo skip installing
```

This will run the command `echo skip installing` for the `install` step.

You can also replace the image defined in the template. Some template steps might rely on commands or environment invariants that your image may not have, so be careful when replacement.

Example:
```yaml
jobs:
    main:
        requires: [~pr, ~commit]
        image: alpine
        template: nodejs/test@1.0.3
```

## Creating a template

Publishing and running templates must be done from a Screwdriver pipeline.

### Writing a template yaml

To create a template, create a new repo with a `sd-template.yaml` file. The file should contain a namespace, name, version, description, maintainer email, and a config with an image and steps. If no namespace is specified, a 'default' namespace will be applied. An optional `images` keyword can be defined to list supported images with a descriptive label. Users can use an alias in place of the actual image (e.g.:, `image: latest-image` would translate into `image: node:8`). Full example in our [screwdriver-cd-test/template-example repo](https://github.com/screwdriver-cd-test/template-example).

Example `sd-template.yaml`:

```yaml
namespace: myNamespace
name: template_name
version: '1.3'
description: template for testing
maintainer: foo@bar.com
images:
    stable-image: node:6
    latest-image: node:8
config:
    image: node:6
    steps:
        - install: npm install
        - test: npm test
    environment:
        FOO: bar
    secrets:
        - NPM_TOKEN
```

### Writing a screwdriver.yaml for your template repo

To validate your template, run the `template-validate` script from the `screwdriver-template-main` npm module in your `main` job to validate your template. This means the build image must have NodeJS and NPM properly installed to use it. To publish your template, run the `template-publish` script from the same module in a separate job.

By default, the file at `./sd-template.yaml` will be read. However, a user can specify a custom path using the env variable: `SD_TEMPLATE_PATH`.

#### Tagging templates
You can optionally tag a specific template version by running the `template-tag` script from the `screwdriver-template-main` npm package. This must be done by the same pipeline that your template is created by. You will need to provide arguments to the script: template name and tag. You can optionally specify a version; the version needs to be an exact version (see `tag` step). If the version is omitted, the most recent version will be tagged (see `autotag` step).

To remove a template tag, run the `template-remove-tag` script. You will need to provide the template name and tag as arguments.

Example `screwdriver.yaml`:

```yaml
shared:
    image: node:6
jobs:
    main:
        requires: [~pr, ~commit]
        steps:
            - install: npm install screwdriver-template-main
            - validate: ./node_modules/.bin/template-validate
        environment:
            SD_TEMPLATE_PATH: ./path/to/template.yaml
    publish:
        requires: [main]
        steps:
            - install: npm install screwdriver-template-main
            - publish: ./node_modules/.bin/template-publish
            - autotag: ./node_modules/.bin/template-tag --name myNamespace/template_name --tag latest
            - tag: ./node_modules/.bin/template-tag --name myNamespace/template_name --version 1.3.0 --tag stable
        environment:
            SD_TEMPLATE_PATH: ./path/to/template.yaml
    remove:
        steps:
            - install: npm install screwdriver-template-main
            - remove: ./node_modules/.bin/template-remove --name myNamespace/template_name
    remove_tag:
        steps:
            - install: npm install screwdriver-template-main
            - remove_tag: ./node_modules/.bin/template-remove-tag --name myNamespace/template_name --tag stable
```

Create a Screwdriver pipeline with your template repo and start the build to validate and publish it.

To update a Screwdriver template, make changes in your SCM repository and rerun the pipeline build.

## Removing a template

### Using screwdriver-template-main npm package
To remove your template, you can run the `template-remove` script. You will need to provide the template name as an argument. 

### Using the UI
Or, you can remove your template and all its associated tags and versions by clicking on the trash icon in the UI on the template page.

_Note: Do not delete your template pipeline beforehand, because it is required to determine who has permission to delete the template._

![Removing](assets/delete-template.png)


