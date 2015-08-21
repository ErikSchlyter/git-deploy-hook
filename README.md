
A Server-Side Git Hook for Automatic Build/Deployment
=====================================================

This script will checkout the newly pushed revision into a temporary directory
and execute a deployment script from it. It can be used to automate test
verification, build your project documentation, update your homepage or
whatever. Personally I push my code to a bare repo on my VPS, which executes
tests and updates the project documentation page before it automatically
forwards the now stable changes to my public GitHub account. The revision will
be rejected if the deployment script exits with a non-zero value.

Install this script by naming it `update` and put it in `yourgitrepo.git/hooks/`

The deployment scripts should be located in the same directory as the git
hooks, and be named by the specific branches they are supposed to deploy
(i.e., `deploy-master`, `deploy-release`, etc.). The script will accept the
revision if there is no deployment script for the given branch.

Source code: <https://github.com/ErikSchlyter/git-deploy-hook>

Download: <http://www.erisc.se/git-deploy-hook/update-checkout_and_deploy.sh>

Example deployment script: <http://www.erisc.se/git-deploy-hook/deploy-master>

This script is based on the `update.sample` script shipped with Git, thereby
licensed under
[GPLv2](http://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html).

### Source code, `update-checkout_and_deploy.sh`
```bash
#!/bin/bash
#
# A Server-Side Git Hook for Automatic Build/Deployment
# =====================================================
#
# This script will checkout the newly pushed revision into a temporary directory
# and execute a deployment script from it. It can be used to automate test
# verification, build your project documentation, update your homepage or
# whatever. Personally I push my code to a bare repo on my VPS, which executes
# tests and updates the project documentation page before it automatically
# forwards the now stable changes to my public GitHub account. The revision will
# be rejected if the deployment script exits with a non-zero value.
#
# Install this script by naming it `update` and put it in `yourgitrepo.git/hooks/`
#
# The deployment scripts should be located in the same directory as the git
# hooks, and be named by the specific branches they are supposed to deploy
# (i.e., `deploy-master`, `deploy-release`, etc.). The script will accept the
# revision if there is no deployment script for the given branch.
#
# Source code: <https://github.com/ErikSchlyter/git-deploy-hook>
#
# Download: <http://www.erisc.se/git-deploy-hook/update-checkout_and_deploy.sh>
#
# Example deployment script: <http://www.erisc.se/git-deploy-hook/deploy-master>
#
# This script is based on the `update.sample` script shipped with Git, thereby
# licensed under
# [GPLv2](http://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html).
#

# Command line
refname="$1"
oldrev="$2"
newrev="$3"

# Usage safety check
if [ -z "$GIT_DIR" ]; then
	echo "Don't run this script from the command line." >&2
	echo " (if you want, you could supply GIT_DIR then run" >&2
	echo "  $0 <ref> <oldrev> <newrev>)" >&2
	exit 1
fi

if [ -z "$refname" -o -z "$oldrev" -o -z "$newrev" ]; then
	echo "usage: $0 <ref> <oldrev> <newrev>" >&2
	exit 1
fi

# Get absolute path of the deployment script (if it exists) for this branch
branch="$(basename $refname)"
deploy="$(realpath "hooks/deploy-$branch")"

# Exit gracefully if we don't have any deployment script for this branch
if ! [ -e "$deploy" ]; then
	echo "No deployment script for $refname, $newrev approved."
	exit 0
fi

# make sure GIT_DIR is a absolute path, since we will execute build scripts from
# a temporary working directory
export GIT_DIR="$(realpath $GIT_DIR)"

# Use a temporary directory as work tree for building/deployment
export GIT_WORK_TREE=$(mktemp -d)

# Check out the new rev into our temporary directory
git checkout -f -q $newrev

# Execute deployment script
if cd $GIT_WORK_TREE && $deploy; then
	# Success - remove temporary directory
	rm -rf $GIT_WORK_TREE
	echo "$deploy successful" >&2
	exit 0
else
	echo "Deployment failed." >&2
	echo "Checked out version available at $GIT_WORK_TREE" >&2
	exit 1
fi

```
### Example deployment script, `deploy-master`
```bash
#!/bin/bash
#
# An example deployment script that creates a README.md and index.html from the
# update-checkout_and_deploy.sh script, and copies it to the path specified by
# `git config deployment.path`.

sed -n -e '2,/^$/s%^# *%%p' ./update-checkout_and_deploy.sh > README.md

echo '### Source code, `update-checkout_and_deploy.sh`' >> README.md
echo '```bash' >> README.md
cat update-checkout_and_deploy.sh >> README.md
echo '```' >> README.md

echo '### Example deployment script, `deploy-master`' >> README.md
echo '```bash' >> README.md
cat deploy-master >> README.md
echo '```' >> README.md

pandoc README.md > index.html

if [ -z "$(git config deployment.path)" ]; then
    echo "No deployment path found. Execute 'git config deployment.path PATH'." >&2
    exit 1
else
    cp * "$(git config deployment.path)/"
fi

exit 0
```
