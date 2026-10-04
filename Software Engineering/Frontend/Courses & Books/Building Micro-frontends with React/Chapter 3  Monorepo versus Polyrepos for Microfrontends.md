---
title: "Chapter 3  Monorepo versus Polyrepos for Microfrontends"
---
There are many strategy/patterns to manage the repos, like Git submodules, Git subtrees, Polyrepos, Monorepos and many more.

Monorepos
    As the name suggests, mono is single and the code is managed in a single Git repository. This means all the team members work on a common single repository that contains multiple applications.
![Untitled 53](/Attachments/Untitled%2053.png)

Polyrepos
    This is where each app has its own repository, Team generally work on multiple repos switching repos as they work on different apps (But ideally, each time work on 1 repo).
![Untitled 54](/Attachments/Untitled%2054.png)


Differences between Monorepo and Polyrepos
    Team Collaboration
        With Polyrepos, teams create their own individual repositories and decide and define their own rules and guidelines on how the code is maintained. This is easy and quick, but has a few drawbacks. Here teams tend to become more isolated and requires to set up and maintain the build pipelines and recommit hooks as duplication on each repo.
        With monorepos, teams are forced to collaborate as the need to find a common way to how the code will be maintained. 
    Build tools and Quality Gates
        With Polyrepos, teams need to implement its own build system and quality gates, this will result in duplication of effort and leads to higher maintenance costs, also it will make the Teams with strong leads way better than Junior teams which will need intervention between teams.
        With monorepos, all build tools and quality gates can be centrally managed, reducing the duplication and we can leverage the strength and skillsets across all teams immediately.
    Code Ownership
        In Polyrepo, the permission are set at the repo level in terms of who has permission to view the code in a repo or make changes to it.
        In Monorepo, all teams have access to view and edit all the files in the code. The permissions and control in monorepo are maintained via a **CODEOWNERS** file, which allows us to set a granular permissions at a folder level.
    Flexibility
        Polyrepos provide the highest level of flexibility, in terms how the code within each team is managed.
        In Monorepo, this flexibility is “Intentionally” restricted to ensure all team members benefit from the best coding practices and tooling setup that the team can offer.
    Refactoring Code
        In Polyrepo, In case you are refactoring code across multiple repos, it can be very time-consuming as you will need to check out all the different repos and individually raise a merge request or pull request for each repository.
        In Monorepo, doing such large-scale refactoring is quite easy by making atomic commits where a single merge request can contain the necessary changes for all apps.
    Ownership of Upgrades
        The time to upgrade libraries or tooling, In Polyrepo teams have the full decision on to take the upgrade or to defer it if they have other priorities. This can be good and bad, as there is always a risk that some teams may get far behind in upgrading their libraries.
        With monorepos, if a shared library or tool is being upgraded, it is easy to make atomic commit across all the apps within monorepo, which means all teams directly get the benefit of the latest versions. 
    Code base Size
        With Polyrepo, your code base gradually increases over time; however with monorepo, you are dealing with a large code base right from day one. and in monorepo it grows exponentially as the application grows.


Popular Monorepo tools:
	1. Lerna (Adopted by NRWL team who built Nx)
		1. This is using packages-based monorepo style.
	2. Nx
		2. This is using integrated monorepo
		3. Supports package based monorepos 
	3. Turborepo
		4. This is using packages-based monorepo style
