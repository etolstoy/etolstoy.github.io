---
layout: post
title: Generamba
description: Generamba is a sweet code generator for iOS projects, supports Objective-C and Swift
---

#### TL;DR

This post is about our [brand new code generator](https://github.com/rambler-ios/Generamba) suited for iOS development. Some of its features are:

- Support for both Objective-C and Swift.
- Powered by liquid template engine.
- Has a flexible template management system.
- Cocoapods integration.

<!--more-->

#### Introduction

Important decisions concerning software architecture often lead to some kind of a compromise. When we implement the [n-tier architecture](https://en.wikipedia.org/wiki/Multitier_architecture), we gain a number of meaningless methods forwarding between upper and lower layers. Decided to implement concurrent heavy lifting - got chronic headache while investigating implicit bugs and deadlocks.

Here at **Rambler&Co** we've faced the same problem when we chose **VIPER** pattern as a common approach to our mobile apps presentation layer design. At one point we've got perfect modularity and testability, but on another - complexity and monotony in new modules creation process.

An average iOS developer creates only one class per screen. But for the one who uses VIPER that's a moment of suffering. To be true - it's a little longer than just a moment. Usually he has to create and fill with boilerplate code for around five classes, six protocols and five test-cases. Let's imagine that our guy managed to hire a professional secretary with an impressive print speed. Even than he would hardly be able to overcome a 30 seconds limit for one file preparation. Multiply these numbers - and you'll get something around 10 minutes per single VIPER module. That average developer from the beginning of the story would already write a couple of hundreds LoC of `UITableViewDataSource`, send some network requests and paint all views in a pretty azure color.

This hard and monotonous manual labor is not the only problem. Another common reason of a developer headache are typos multiplying with each new module.

#### Using Xcode Templates

One of possible solutions of the problem are Xcode templates. We used this approach to code generation for a rather long time, and made a list of its drawbacks:

- Xcode is definitely not the most stable IDE. Sometimes templates and plugins may break after version update.
- There is no handy way of adding new templates or installing updates of existing ones.
- It's literally impossible to link one generated file to multiple targets.
- The syntax is extremely ugly and unfriendly especially for a newcomer whose only desire is to create a simple template for automating his tasks.

And the last and most fatal drawback is that this tool doesn't belong to us. We don't have enough control over its behavior, and the extensibility for more complex tasks approaches zero.

#### What Generamba is

That were the reasons to create our own code generator called Generamba. We got a highly extensible tool for a wide range of different code generation tasks though originally it was developed with just VIPER modules in mind.

![Generamba Screenshot](/public/img/posts/generamba.jpg)

You have to complete three steps from Generamba install to a first module generation:

- `generamba setup` command configures Generamba for a selected project. You specify default file paths for code and tests, dependency managers settings and other infrastructure options at that moment. You'll get a Rambafile as the result. It's the main configuration file of the current project.
- `generamba template install` installs templates specified in the *Rambafile*.
- `generamba gen BrandNewModule rviper_controller` directly creates a new module called `BrandNewModule` using a `rviper_controller` template.

Generamba uses three types of resources:

- *Rambafile* is a configuration file that contains a data inputted during generamba setup command, templates names and template catalogs paths.

```yml
### Headers settings
company: Rambler&Co

### Xcode project settings
project_name: GenerambaSandbox
prefix: RDS
xcodeproj_path: GenerambaSandbox.xcodeproj

### Code generation settings section
# The main project target name
project_target: GenerambaSandbox

# The file path for new modules
project_file_path: GenerambaSandbox/Classes/Modules

# The Xcode group path to new modules
project_group_path: GenerambaSandbox/Classes/Modules

### Tests generation settings section
# The tests target name
test_target: GenerambaSandboxTests

# The file path for new tests
test_file_path: GenerambaSandboxTests/Classes/Modules

# The Xcode group path to new tests
test_group_path: GenerambaSandboxTests/Classes/Modules

### Dependencies settings section
podfile_path: Podfile
cartfile_path: Cartfile

### Templates
catalogs:
- 'https://github.com/rambler-ios/generamba-catalog'
- 'https://github.com/igrekde/my-own-catalog'
templates:
- {name: rviper_controller}
- {name: local_template_name, local: 'absolute/file/path'}
- {name: remote_template_name, git: 'https://github.com/igrekde/remote_template'}
```

- One or more templates that can be used for generamba gen command.
- The current user settings, e.g. his name.

The first two points can be safely stored in a Git repository, but the last one is located in a project-independent directory.

#### Templates

One of the most important Generamba advantages is its templates support. Unlike many other code generators we don't ship with preinstalled templates. You are free to use a flexible template management system similar to some dependency managers (e.g. Cocoapods). New templates can be installed via:

- A local path (the template is copied from a specified directory).
- A remote repository URL (the template is cloned).
- A template catalog - either shared or [custom](https://github.com/rambler-ios/generamba-catalog).

A common pattern of Generamba usage is a dedicated template catalog for a project which contains all of the templates. Some of them after a short debate make their way to a team's main catalog.

One of the reasons of forbidding Xcode templates was their markup. In contrast, Generamba is powered by [Liquid template engine](https://github.com/Shopify/liquid). It's well known for its simple, clear syntax and a lot of handy features: loops, partials, conditionals and so on.

**Xcode template example:**

```objc
//
//  ___VARIABLE_viperModuleName______FILENAME___
//  ___PROJECTNAME___
// 
//  Created by ___FULLUSERNAME___ on ___DATE___
//  Copyright ___YEAR___ ___ORGANIZATIONNAME___. All rights reserved.
//

#import "___VARIABLE_viperModuleName:identifier___Interactor.h"
#import "___VARIABLE_viperModuleName:identifier___InteractorOutput.h"

@implementation ___VARIABLE_viperModuleName:identifier___Interactor

#pragma mark - ___VARIABLE_viperModuleName:identifier___InteractorInput

@end
```

**Generamba template example:**
{% raw %}
```objc
//
//  {{ module_info.name }}{{ module_info.file_name }}
//  {{ module_info.project_name }}
//
//  Created by {{ developer.name }} on {{ date }}.
//  Copyright {{ year }} {{ developer.company }}. All rights reserved.
//

#import "{{module_info.name}}Interactor.h"
#import "{{module_info.name}}InteractorOutput.h"

@implementation {{ module_info.name }}Interactor

#pragma mark - {{ module_info.name }}InteractorInput

@end
```
{% endraw %}
#### Future Plans

We keep on evolving Generamba. Besides common tasks we look at some major areas of improvement:

- GUI including IDE plugins,
- Android support,
- Science fiction requiring AST processing for generating test cases or mocks based on classes and protocols declarations.

In conclusion I'd like to mention that Generamba is an obvious case of task automation benefit. I'll use this experience in other areas of activity requiring manual labor.

Feel free to submit proposals and questions in our GitHub repository - Generamba community is small but responsive.

#### Links

- [liftoff](https://github.com/thoughtbot/liftoff) - Xcode projects generation,
- [The Book of VIPER](https://github.com/rambler-ios/The-Book-of-VIPER) - available only in Russian,
- [Generamba Documentation](https://github.com/rambler-ios/Generamba/wiki).