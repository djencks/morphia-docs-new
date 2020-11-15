+++
[menu.main]
  identifier = "Issues and Help"
  weight = 40
  pre = "<i class='fa fa-book'></i>"
+++

# Issues & Help

We are lucky to have a vibrant MongoDB Java community with lots of varying
experience of using Morphia.  We often find the quickest way to get support for
general questions is through the [Morphia google group](https://groups.google.com/forum/#!forum/morphia),
[the mongoDB community forums](https://community.mongodb.com/c/drivers-odms-connectors/),
or through [stackoverflow](https://stackoverflow.com/questions/tagged/morphia).

## Bugs / Feature Requests

If you think you’ve found a bug or want to see a new feature in the Morphia, please open an issue on
[github](https://github.com/MorphiaOrg/morphia/issues). Please provide as much information as possible (including version numbers) about the 
issue type and how to reproduce it.  Ideally, if you can create a reproducer for the issue at hand, that would be even more helpful.  To
help with this, please take a look at the [reproducer](https://github.com/MorphiaOrg/reproducer) project.  This will help you set up a
quick environment for reproducing your issue and providing a working example to examine.  This project can either be shared via a github
repo on your account or perhaps attaching a zip of the project to the associated issue.

## Pull Requests

We are happy to accept contributions to help improve Morphia.  We will guide user contributions to ensure they meet the standards of the 
codebase. Please ensure that any pull requests include documentation, tests, and also pass the build checks.

To get started check out the source and work on a branch:

```bash
$ git clone https://github.com/MorphiaOrg/morphia.git
$ cd morphia
$ git checkout -b myNewFeature
```

Finally, ensure that the code passes all the checks.
```bash
$ cd morphia
$ mvn -Pquality
```
