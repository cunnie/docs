You almost always have to do this:
```
git config --global user.name "Brian Cunnie"
git config --global user.email brian.cunnie@gmail.com
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.st status
git config --global url."git@github.com:".insteadOf "https://github.com/"
```
For systems with no defaults, turn on colors
```
git config --global color.diff auto
git config --global color.status auto
git config --global color.branch auto
git config --global core.editor nvim
```
On FreeBSD, `less` should allow escape sequences
```
git config --global core.pager 'less --raw-control-chars'
```
Dmitriy hacks:
```
git log origin/256.x...origin/257.x --oneline --stat|grep stemcell_builder
git show origin/256.x...origin/257.x -- stemcell_builder/stages/image_ovf_vmx/apply.sh
git show origin/256.x...origin/257.x --oneline --stat -- stemcell_builder
git show origin/256.x...origin/257.x  -- stemcell_builder
```
make sure you already have the password for user koda saved
try "svn co https://svn.pivotallabs.com/subversion/koda kodajunk"
```
git svn clone -s https://svn.pivotallabs.com/subversion/koda --username koda
git svn clone http://ie7-js/goodlecode.com/svn/trunk
```
misc:
```
cd /usr/local/etc/asterisk
git init
git add .
git commit
#
echo "*" > .gitignore
git add -f .gitignore
git add -f `ls /var/qmail/control/* | grep -v '.cdb'`
# 
git remote add origin git@github.com:briancunnie/asterisk.git
git push -u origin master # only need -u for initial checkin
#
cd /etc/asterisk
sudo git init --shared=group
sudo git add .
git commit
# to make an asterisk repository
git clone --bare /etc/asterisk ~it/git/asterisk
touch ~it/git/asterisk/git-daemon-export-ok
# to make a repository on a non-nfs-mounted machine (tata)
sudo git clone --mirror samstag:/etc/ ~it/git/samstag-etc
sudo git clone --mirror samstag:/usr/local/etc/ ~it/git/samstag-usrlocaletc
sudo touch ~it/git/samstag-{etc,usrlocaletc}/git-daemon-export-ok
# more stuff
git clone git://github.com/thoughtworks/cruisecontrol.rb.git ~/ccrb
git log
git branch pivotal_branch
git checkout pivotal_branch
git fetch --tags origin
git branch -a
git checkout origin/1.4.0
cd ~/ccrb; echo "old version works" | git checkout 47adcb8dff3a1816938029a30c258443e5abd17b
#git push ssh://git.ardatech.com/~it/git/asterisk master:master
cd /etc/asterisk
git push ~it/git/asterisk 
git add -A; sudo git commit -a
# 
git clone git://github.com/thoughtworks/cruisecontrol.rb.git ~/ccrb
# ccrb 1.4.0+ is broken, we need to revert to 1.4.0 pristine
cd ~/ccrb; echo "old version works" | git checkout origin/1.4.0
sudo vim .gitignore
  ipp.txt
  openvpn-status.log
sudo git rm --cached ipp.txt openvpn-status.log
# to check out submodules
git submodule add tata:~it/git/samstag-usrlocaletc etc
# to check out sparse
git config --local core.sparseCheckout "true"
# to do git diff
git diff head^^ recipes/lion_basedev.rb
# to delete a branch lion
git branch -d lion
# to delete a remote branch (i.e. on origin)
git push origin :lion
# merging a pull request
git remote add house9 https://github.com/house9/pivotal_workstation.git
git fetch house9
git checkout -b house9 house9/master
git checkout master
git merge house9
# git branch -a #to see the other branches
# to see what files you should have deleted but didn't do a git 'rm'
git clean
git clean -f
# checkout a file you deleted in a previous commit
git checkout HEAD^ file_you_deleted

# doing a partial merge of a pull request
git checkout master
# Check out your master branch
git remote add outright git://github.com/outright/pivotal_workstation.git -t gridapp
# Add a new remote named 'outright'
git fetch outright
# Pull in all the commits from the 'outright' remote
git cherry-pick outright/gridapp
# Merge your master branch into the 'my-branch' branch from the 'outright' remote
git push origin master# Push your newly-merged branch back to GitHub

git log --graph --pretty=format':%C(yellow)%h%Cblue%d%Creset %s %C(white) %an, %ar%Creset'
cd /; sudo git submodule add -f git@github.com:pivotalops/scripts.git /usr/local/workspace/scripts
cd /; sudo git config -f .git/config --remove-section submodule./usr/local/workspace/scripts
cd /; sudo git config -f .gitmodules --remove-section submodule./usr/local/workspace/scripts
git config --global alias.st status

git remote add cunnie https://github.com/cunnie/sprout
git branch --track pivotal_logos cunnie/pivotal_logos

git config diff.tool vimdiff
git difftool bosh_create_dns_server.md
```
