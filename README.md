Drush Subtree
=============

Contents
--------

 - [Overview](#overview)
 - [Dependencies](#dependencies)
 - [Usage](#usage)
 - [Build Manager integration and user stories](#build-manager-integration-and-user-stories)
   - [As a maintainer of contributed modules and themes...](#as-a-maintainer-of-contributed-modules-and-themes)
   - [As a maintainer of a site built on somebody else's distro...](#as-a-maintainer-of-a-site-built-on-somebody-elses-distro)
   - [As a maintainer of a contributed distro...](#as-a-maintainer-of-a-contributed-distro)
 - [Understanding how this all works with Drush Make](#understanding-how-this-all-works-with-drush-make)
   - [Drush Make build files](#drush-make-build-files)
   - [Drush Make and recursion](#drush-make-and-recursion)
   - [Project versions should be declared in toplevel build file](#project-versions-should-be-declared-in-toplevel-build-file)
   - [Distro maintainers: Do not store subtrees inside subtrees](#distro-maintainers-do-not-store-subtrees-inside-subtrees)

Overview
--------

Drush Subtree's goal is to improve Drupal developers' ability to
make their work reusable. Specifically, it simplifies workflows for developing
and maintaining contributed projects by enabling you to do development on
contrib projects from inside a parent site repository. It does this by providing
a wrapper around git-subtree to help you manage Git repos
inside a parent repo and by providing integration with Drush's Build
Manager extension for automating builds and using subtrees with Drush Make.

  - Use Build Manager's interactive prompt to set your site up with Git subtrees
    for projects you maintain outsite the parent site repository.

  - Do development inside whatever site repo you're actively working in,
    then push commits out of your site repo up to drupal.org or (whatever
    private repo your custom module or theme lives in).

  - Pull updates into your site repo from an outside repo for your
    distro, module, theme, or custom project.

  - For teams working together on a site repository, this enables team
    members to contribute to a project inside a site repo, through your own
    internal workflow. Then it enables you--the maintainer--to easily push that
    work out to public repos when it's ready to be released.

  - Incorporate Git subtrees into Drush Make builds for your site repository.
    (Provides support for checking out tagged versions of subtree projects without
    providing a specific commit ID.)


Dependencies
------------

  - [git-subtree](https://github.com/git/git/tree/master/contrib/subtree)
  - [Build Manager](https://github.com/whitehouse/buildmanager) - Note: even if you
    don't use Build Manager to manager Drush Make builds, Drush Subtree expects
    you to store info about your subtrees in a Build Manager configuration file.
  - (Recommended) Drush master / 7.x (Build Manager uses the Drush Make
    --no-recursion flag and autoloading which--as of the time of this writing--
    ave not been backported to 6.x)

Usage
-----

Store info about your site repository's subtrees in a Build Manager
configuration file. Use the interactive prompt to have Build Manager generate
this file for you, or see examples included with Drush Subtree to create the file manually.

Start the interactive prompt like this:

    drush buildmanager-configure

Next you can manually add/update subtrees with the commands described below, or via automated
Build Manger builds. (See [Build Manger integration](#build-manager-integration)
for information on how Drush Subtree works with automateds builds managed
by Drush Make and Build Manager.)

Here's a quick overview of the main commands:

    # Note: All the commands below are pushing to or pulling from a remote
    # repository specified in your Build Manager config file.

    # Add a subtree to the parent repository.
    drush subtree add <project>                

    # Pull updates in from outside your site repo.
    drush subtree pull <project>               

    # Push updates out to external repo from inside your site repo.
    drush subtree push <project>               

    # "Checkout" a tagged version of a subtree project (e.g. 7.x-2.1) inside your
    # site repository. Note: This is a faux subtree command. (It only exists via
    # Drush Subtree, there is no `git subtree checkout` command.)
    drush subtree checkout <project> <tag>

    # Specify a particular commit ID to use or "checkout" for a subtree project.
    drush subtree merge <project> <id>

For more details and examples, see:

    drush subtree --help

If you're not familiar with git-subtree, you may prefer the more Drush-y
interface, *drush subtree-\<verb\> \<args\>*, see:

    drush --filter=drushsubtree

Tips for getting started:

  - To see what's going on with git under the hood, use Drush's verbose flag `-v`
  - To examine shell commands generated by Drush Subtree without running them,
    use Drush's `--simulate` flag.
  - Don't include changes to multiple projects (i.e. two different modules) all
    in the same commit. For a project stored in a subtree, this enables you to push
    changes out cleanly. For projects that are NOT stored in a subtree, this
    will make it easy for you to break custom projects out of your site repo and
    make them stand-alone projects later, without losing your commit history.
  - To keep things light, clean, and simple: Only add subtrees to your code base
    for projects (or forks) you maintain or actively work on. Use vanilla Drush
    Make downloads for everything else.
  - Subtree add/pull/merge commits don't like to be rebased. Avoid running these
    commands in the middle of active development, during a part of your
    project's history that's likely to be rewritten before you publish.


Build Manager integration and user stories
------------------------------------------

Drush Subtree integrates with Build Manager to provide a few niceties:

  - Hooks into the `drush buildmanager-configure` interactive prompt to help you
    set up and manage subtrees
  - Hooks into `drush buildmanager-build` to figure out which projects
    referenced in your make file(s) are subtrees and manage them accordingly
    during your automated (re)builds

See relevant user stories below to find out more about how you can use this.

### As a maintainer of contributed modules and themes...
**...I want to be able to work on my project inside my site repo. But I also
want to be able to pull in updates, when they're coming from outside my site
repo.**

TODO


### As a maintainer of a site built on somebody else's distro...
**...I want it to be fast and easy to pull in updates, test, and deploy.**

I organize my site repo as described below.

Include the distro named _exampledistro_ in my site repo as a subtree by including the project in _buildmanager.config.yml_ like this:

        subtrees:
          exampledistro:
            path: docroot/profiles/exampledistro
            uri: http://git.drupal.org/project/exampledistro.git
            branch: 7.x-1.x
            squash: true
            message: exampledistro subtree from http://git.drupal.org/project/exampledistro.git
          
I make the distro's build file the base of my own build file by writing my _build.make_ like this:

        ; Include exampledistro's build file.
        includes[base] = projects/exampledistro/build-exampledistro.make

        ; Site-specific overrides and patches to contrib projects included by exampledistro.
        projects[some-project][patch][12345] = https://drupal.org/files/issues/12345-some-issue-1.patch

        ; Add any additional contrib or custom projects included in my site, but
        ; not included with the distro I'm using.
        projects[my-project1][version] = 7.x-3.0
        projects[my-project2][version] = 7.x-2.2

When I (re)build with `drush buildmanager-build` my repo's directory structure
looks like this:

        buildmanager.config.yml                  # Build Manger config, stored at toplevel of repo
        build.make                               # My build file, stored at toplevel of repo
        projects/exampledistro                   # Subtree of distro
        default                                  # Includes my settings.php, files directory, etc.
        docroot                                  # Drupal codebase
        docroot/sites/default -> ../../default   # Symlink to default directory
        docroot/profiles/exampledistro           # This is a download of the latest stable release declared
                                                 # in projects/exampledistro/build-exampledistro.make
        docroot/sites/all/modules/my-project1    # Contrib projects included by you in build.make go here
        docroot/sites/all/modules/my-project2    
        docroot/sites/all/modules/some-project   # Patched version of some-project, overriding what's included
                                                 # by default with distro.


### As a maintainer of a contributed distro...
**...I want to**


Understanding how this all works with Drush Make
-------------------------------------------------

### Drush Make build files

Drush Subtree stores project subtrees outside your Drupal
codebase, then includes them via symlinks. This means, it's safe for you to
include a make file from a subtree project in your toplevel build file.


### Drush Make and recursion

Drush Make recursively discovers and includes
make files when a parent make file downloads another project with it's own make.
Including new make files in the build recipe at runtime raises a few issues you
should be aware of when working with subtrees.

#### Project versions should be declared in toplevel build file

When you (re)build a site codebase with buildmanager-build, Drush
Subtree only finds version information (release tags) available in your toplevel
make file (e.g. build.make) or in a make file explicitly incuded in the toplevel
make file. Drush Subtree does NOT automatically manage versions of subtree
projects discovered by Drush Make at runtime.

Recommendation: Production builds including subtrees should include
projects with release versions in the toplevel make file (or makefiles
explicitly included by it).

Alternative: If the recommended practice doesn't work for you for any reason you
can either (1) accept the fallback behavior, projects will run on the tip of
whatever branch is specified in your buildmanager.config.yml, or (2) you can add
your own custom workflow to buildmanager.config.yml as prebuild or postbuild
commands.

To see all the make files in your code base you can run:

    drush buildmanager-find-make-files
    drush bmfmf

An easy way to see all the project info included in your toplevel make file is
to run this command:

    drush buildmanager-build --simulate --show-info

#### Distro maintainers: Do not store subtrees inside subtrees

_Issue #2_: If you maintain an install profile and a contrib project required by
it, a common directory structure is to include the contrib project _inside_ the
profile for example: docroot/profiles/my-profile/modules/contrib/my-module.
Subtrees inside subtrees don't work (well, technically they "work", but it's a nasty mess of a sitution
and Drush Subtree does not support it). You have a few alternatives:

A. Rather than include multiple make files in your distro, put everything in
   build-mydistro.make (remove drupal-org.make and drupal-org-core.make). Then Drush Make will put included projects in sites/all
   rather than inside profiles/my-profile. (Rather than put contrib projects in
   sites/all/contrib you might put them in sites/all/my-profile to make clear to
   users that these are projects required by your distro.) This is a nice solution because now
   you and all the people who use your distro will have the same directory
   structure, which just keeps things simple and easy to understand.

B. You can leave all your make files as they are, then include as second
   copy of my-module in sites/all/modules. Drupal will give this module priority
   and load this one. To do this, make a simple build.make file that looks like
   this:

        includes[base] = projects/my-profile/build-myprofile.make
        projects[] = my-project

C. Turn off recursion.

  - TODO note on --no-recursion
 - Drush Subtree assumes you are using the `--no-recursion` flag with your builds. If
you do NOT use `--no-recursion` Drush Subtree will not be able to detect subtrees
and versions included make files that are added during runtime, this would lead
to inconsistent and confusing results.

**Re. Distros**: You can't store subtrees inside subtrees. If you maintain a
distro AND maintain any contrib projects included in that distro, don't organize
your directories like this:

    my-repo/docroot/profiles/my-profile/modules/contrib/my-module
    <parent-repo>/docroot/profiles/<subtree>/modules/contrib/<subtree>
 
Instead, do this:

    # Only store your profile project (and any included custom modules that live
    # in that same repo) here:
    my-repo/docroot/profiles/my-profile

    # Contrib projects go in sites/all like my-module here:
    my-repo/docroot/sites/all/modules/contrib/my-module

    # If you want to make clear in the build, which contrib projects are being
    # included by the distro, name the directory containing all the contrib
    # projects after your profile, rather than "contrib", like this:
    my-repo/docroot/sites/all/modules/my-profile/my-module

To get drush make to build your code base ^^ this way, you can either (a) move
the contents of drupal-org.make into your build-myprofile.make file, or (b)
create your own build.make with includes referencing build-myprofile.make and
drupal-org.make and pass drush make the --no-recursion flag.
