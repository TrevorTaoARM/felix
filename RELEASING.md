# Felix 1.4 branch release process

## Prerequisites

Get (or update to) the latest code from
https://github.com/tigera/process, in <process_dir>.  This repo
contains scripts to automate some of the process.

## Prepare Felix 1.4.x release candidate

To prepare a Felix 1.4.x release candidate, get a clone of
https://github.com/projectcalico/felix master, and in the root of that
clone:

- Run ‘utils/create-release-commit.sh <version>’.

- When prompted, review the new section that has been automatically
  added at the top of CHANGES.md, and update it if necessary so as to
  be a good description for outside readers of what is new or changed
  in the new release.

- When happy with the content of CHANGES.md, hit Return to allow
  create-release-commit.sh to continue.  (It propagates the new
  version and description of changes to the other places that they are
  needed, and makes a local Git commit with all the changes so far.)

- Get the release commit reviewed, but don’t push or merge it yet.
  (In case it needs tweaking, it’s cheaper to do that now before all
  the package building and testing.)

- Use ‘dpkg-buildpackage -S -kCalico’ and ‘dput ...’ to build source
  packages for Xenial and upload the release candidate Debian source
  packages them to a release testing PPA.

- Delete the top stanza from debian/changelog (which has a version
  with ‘-xenial’) and do that again, to build and upload packages for
  Trusty.

- Run ‘<process_dir>/build-rpm.sh’ to build RPMs - which will appear
  in the ‘rpm’ directory.

## Publish candidate RPMs to testing repository on binaries.projectcalico.org

If there isn’t already a testing repo on binaries, create it by taking
a copy of the last release repo.  If a testing repo already exists, it
should be identical to the last release repo.

    gcloud config set project tigera-wp-tcp-redirect
    gcloud compute ssh ubuntu@binaries-projectcalico-org --zone us-east1-c
    cd /usr/share/nginx/html/rpm
    diff -ur calico-1.4 calico-1.4-testing # to check an existing testing repo
    cp -a calico-1.4 calico-1.4-testing # if testing repo does not already exist

Copy release candidate RPMs to the testing repo on binaries

    gcloud compute copy-files rpm/*.noarch.rpm \
    ubuntu@binaries-projectcalico-org:/usr/share/nginx/html/rpm/calico-1.4-testing/noarch/ --zone us-east1-c

Run ‘createrepo .’ in the repo on binaries.

    gcloud compute ssh ubuntu@binaries-projectcalico-org --zone us-east1-c
    cd /usr/share/nginx/html/rpm/calico-1.4-testing
    createrepo .

## Release testing

Once release candidate packages have been built and published, run
tests against the testing PPA/repo.

## Good to release?

Publish packages to the release PPA and RPM repo.

- For PPA packages, use Launchpad to copy from calico-1.4-testing to
  calico-1.4.

- For RPMs, on binaries, sync from calico-1.4-testing to calico-1.4.

    gcloud compute ssh ubuntu@binaries-projectcalico-org --zone=us-east1-c
    cd /usr/share/nginx/html/rpm/calico-1.4
    rsync -av --delete ../calico-1.4-testing/* ./

Tag and publish the Calico release.  (Note that this will send
announcements to people subscribed to GitHub.)

- Merge the release commit.

- Create the PyInstaller bundle by running the build-pyinstaller
  Jenkins job using "Build with parameters", specifying the commit ID
  of the proposed release.  It will attach the
  calico-felix-<version>-git-<hash>.tgz to the job as an artefact.

- git tag <version>

- git push --tags

- At https://github.com/projectcalico/felix/tags, click ‘Add release
  notes’.

  - In the main edit box, paste in the same release summary as in
    debian/changelog.

  - Upload the pyinstaller bundle as a release binary.

  - Then click the “Publish release” button.

- After the release is tagged, make a new commit on master that revs
  the version numbers so that new packages built from master will be
  pre-releases of the next release rather than looking like dupes of
  the release.  (If we need to make a bugfix release, we take a branch
  from the release tag rather than cutting from master.)