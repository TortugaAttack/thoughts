# CI using maven and mkdocs for documentation

This will setup your github pages, creating automatically releases and publishes packages as soon as you merge into `main` 
The website will then contain the following assuming the version in your pom is MAJOR_MINOR_VERSION_IN_POM: 
* https://YOUR_GROUP.github.io/YOUR_PROJECT_NAME/MAJOR_MINOR_VERSION_IN_POM/ - the documentation 
* https://YOUR_GROUP.github.io/YOUR_PROJECT_NAME/javadoc/MAJOR_MINOR_VERSION_IN_POM/apidocs - the javadoc

Be aware that this uses the versioning major.minor.build and updates the javadoc and documentation if a major or minor update will happen, as a build version should generally not change the documentation.
Be aware that even if you just release a build update the javadoc and documentation will be overwritten, allowing minor changes such as correcting words/grammar.

## Restricting main 

In your project go to settings->Branches and add a Branch protection rule for your main branch (this will be the release branch)
Set 
* Require pull request reviews before merging
* Require status checks to pass before merging 
* Include administrators 

This will restrict from accidentally pushing directly to the main branch, which would accidentally releasing.

## Setting up Github Pages. 

Create the `gh-pages` branch in your project

In your project go to settings->Pages and set the Source to gh-pages. 

## MKDocs 

If you don't have a mkdocs documentation create one in  your project
```
mkdocs new . 
```

edit mkdocs.yml and exchange everything starting with YOUR. (Note the $VERSION and $RELEASE_VERSION will be automatically exchange later on) 

```yaml
extra:
  version: $VERSION
  release_version: $RELEASE_VERSION
  social:
    - icon: fontawesome/brands/github
      link: YOUR_GITHUB_LINK

site_name: YOUR_PROJECT_NAME 


edit_uri: ""
theme:
  name: material
  features:
    - navigation.tabs
    - navigation.top
    - toc.integrate

  include_search_page: false
  search_index_only: true

  language: en
  font:
    text: Roboto
    code: Roboto Mono
  favicon: assets/favicon.png
  icon:
    logo: logo
    repo: fontawesome/brands/git-alt
  palette:
    - media: "(prefers-color-scheme: light)"
      scheme: default
      toggle:
        icon: material/toggle-switch-off-outline
        name: Switch to dark mode
    - media: "(prefers-color-scheme: dark)"
      scheme: slate
      toggle:
        icon: material/toggle-switch
        name: Switch to light mode

repo_url: YOUR_REPO_URL
repo_name: YOUR_GROUP/YOUR_PROJECT_NAME

plugins:
  - macros
  - search

```

Everywhere in your documentation where you want to use the version simply use `{{ version }}` it will be exchanged with the version in your pom at deployment.

Now you'll get a nice and beautiful documentation using mkdocs-material and you just need to create your documentation (see. https://www.mkdocs.org/)

## setting up the pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  ...
  <version>${major.minor.version}.${build.version}</version>
  <properties>
        <major.minor.version>${major.version}.${minor.version}</major.minor.version>
        <major.version>YOUR MAJOR VERSION</major.version>
        <minor.version>YOUR MINOR VERSION</minor.version>
        <build.version>YOUR BUILD VERSION</build.version>
    ...
  </properties>
  
...
    <distributionManagement>
        <repository>
            <id>github</id>
            <name>GitHub Packages</name>
            <url>https://maven.pkg.github.com/YOUR_GROUP/YOUR_PROJECT_NAME</url>
        </repository>
    </distributionManagement>
...

   <plugins>
    <plugin>
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-javadoc-plugin</artifactId>
       <configuration>
          <reportOutputDirectory>site/javadoc/${major.minor.version}/</reportOutputDirectory>
       </configuration>
    </plugin>
   </plugins>
```

This will setup the package managment using github and creating the javadoc and setting that to /javadoc/VERSION/apidocs 
If you don't want the apidocs add `<dirDest></dirDest>` to the configuration as well.

### Setting the shaded jar 

Assuming you're using the maven shaded plugin 
ensure that the shaded jar will not deploy as follows
```xml
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.1</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <shadedArtifactAttached>true</shadedArtifactAttached>
                            <shadedClassifierName>shaded</shadedClassifierName>
                            ...
                        </configuration>
                    </execution>
                </executions>
            </plugin>

```

## Add index.html to redirect to the latest documentation 

add index.html to your repo with the following content.

```html
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="refresh" content="5; url='https://YOUR_GROUP.github.io/YOUR_PROJECT_NAME/$VERSION'" />
  </head>
  <body>
  redirecting to newest documentation... Click <a href="https://YOUR_GROUP.github.io/YOUR_PROJECT_NAME/$VERSION" >here</a>  if nothing happens.
  </body>
</html>

```

## Adding GitHub check to assure that version doesn't exist yet

Add a script to `.github/scripts/tagexists.sh` and `chmod +x .github/scripts/tagexists.sh`
Add the following: 

```bash
#!/bin/sh

if git rev-parse "$1" >/dev/null 2>&1; then
  echo "Tag $1 exist - update version in pom!"
  exit 1
else
  echo "Tag $1 does not exist - good to go."
  exit 0
fi
```

Now create the check by creating a file at `.github/workflows/lint.yml`

```yml
name: lint
on: pull_request

jobs:
  lint:
    name: Release Tag existence Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@main
      - run: .github/scripts/tagcheck.sh v$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
```

This should assure that the PR won't happen until the version is properly set new.

## Create CHANGELOG.md for release body

Create a file called CHANGELOG.md and write all changes since the last version in there. 
This will be used as the text body in the release.

OPTIONAL:
Add to the CHANGELOG.md
```
# New Features

# Issues

# Documentation

```

and setup the following workflow at `.github/workflows/issues.yml`

```yaml
on:
  issues:
    types: [closed, reopened]

jobs:
  test:
    name: issue worker
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          ref: 'develop'
      - name: closed bug
        if: ${{ github.event_name == 'issues' && github.event.action == 'closed'  &&  contains(github.event.issue.labels.*.name, 'bug') && !contains(github.event.issue.labels.*.name, 'wontfix') && !contains(github.event.issue.labels.*.name, 'invalid') }}
        shell: bash
        run: perl -p -i -e "s/Issues\n/Issues\n\n\* \#${{ github.event.issue.number }} - ${{ github.event.issue.title }} /"  CHANGELOG.md
      - name: closed documentation issue
        if: ${{ github.event_name == 'issues' && github.event.action == 'closed'  && contains(github.event.issue.labels.*.name, 'documentation') && !contains(github.event.issue.labels.*.name, 'wontfix') && !contains(github.event.issue.labels.*.name, 'invalid') }}
        shell: bash
        run: perl -p -i -e "s/New Features\n/New Features\n\n\* \#${{ github.event.issue.number }} - ${{ github.event.issue.title }} /"  CHANGELOG.md
      - name: closed feature
        if: ${{ github.event_name == 'issues' && github.event.action == 'closed'  && contains(github.event.issue.labels.*.name, 'enhancement') && !contains(github.event.issue.labels.*.name, 'wontfix') && !contains(github.event.issue.labels.*.name, 'invalid') }}
        shell: bash
        run: perl -p -i -e "s/Documentation\n/Documentation\n\n\* \#${{ github.event.issue.number }} - ${{ github.event.issue.title }} /"  CHANGELOG.md
      - name: reopend
        shell: bash
        if: ${{ github.event_name == 'issues' && github.event.action == 'reopened'   && ( contains(github.event.issue.labels.*.name, 'bug') || contains(github.event.issue.labels.*.name, 'documentation') || contains(github.event.issue.labels.*.name, 'enhancement')) }}
        run: perl -p -i -e "s/\* \#${{ github.event.issue.number }} - [^\n]*\n//" CHANGELOG.md
      - uses: EndBug/add-and-commit@v7 # You can change this to use a specific version
        with:
          add: 'CHANGELOG.md'
          branch: develop
```

This workflow will add Issues wich are labeled as bug, documentation or enhancement to the corresponding List in the CHANGELOG.md
and thus the changelog never has to be touched manually. 
It will be commited to develop, which would mostly support a workflow where an Issues is closed only if the feature/bug fix is already merged into `develop`.


## Continous Integration Workflow

create a file called ci.yml in .github/workflows/ 
```yaml
name: ci
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          java-version: '11'
          distribution: 'adopt'
      - uses: actions/setup-python@v2
        with:
          python-version: 3.x
      - shell: bash
        run: mvn help:evaluate -Dexpression=major.minor.version -q -DforceStdout > version.log
      - shell: bash
        run: mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout > artifactid.log
      - name: Set env version
        run: echo "RELEASE_VERSION=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)" >> $GITHUB_ENV
      - name: Set env name
        run: echo "RELEASE_ARTIFACTID=$(cat artifactid.log)" >> $GITHUB_ENV  
      - name: test
        run: echo ${{ RELEASE_VERSION }} ${{ RELEASE_ARTIFACTID }}
      - run: pip install mkdocs-material
      - run: pip install mkdocs-macros-plugin
      - run: sed -i "s/\$VERSION/$(cat version.log)/g" mkdocs.yml
      - run: sed -i "s/\$RELEASE_VERSION/${{ RELEASE_VERSION }}/g" mkdocs.yml 
      - run: mkdocs build -d site/$(cat version.log)
      - run: mvn javadoc:javadoc
      - run: sed -i "s/\$VERSION/$(cat version.log)/g" index.html
      - run: cp index.html ./site
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./site
          destination_dir: ./
      - name: Publish package
        run: mvn --batch-mode deploy
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Create Release
        id: create_release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ RELEASE_VERSION }}
          release_name: Version ${{ RELEASE_VERSION }}
          draft: false
          prerelease: false
          body_path: CHANGELOG.md
      - uses: actions/upload-release-asset@v1.0.1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }}
          asset_path: ./target/${{ RELEASE_ARTIFACTID }}-${{ RELEASE_VERSION }}-shaded.jar
          asset_name: ${{ RELEASE_ARTIFACTID }}-${{ RELEASE_VERSION }}.jar
          asset_content_type: application/zip

```

This will automatically get the artifactID and version from the pom .xml 
then will exchange $VERSION in  mkdocs and create the documentations (in this version allows material a very nice theme for mkdocs).
Further on creates the javadoc for the maven project. Both will be published to the gh-pages branch.

Afterwards the package will be deployed to github packages and a release with the version set in the pom.xml will be released including the final shaded jar.

And thats it. 
As soon as you PR into main it will create and version the documentation as well as javadoc into $VERSION/ and javadoc/$VERSION/apidocs and creates a fully release and publishing a maven package.
