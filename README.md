# www.davidwang.com
A Github-Hosted statically generated Hugo Minimal Blog and Personal Website

```
brew install hugo
make develop
```


Current version of Blackburn as of 20200812 has a bug where social icons don't show.
  https://github.com/yoshiharuyamashita/blackburn/pull/101

Until the PR merges

```
cd themes/blackburn/layouts/partials
rm share.html
wget https://raw.githubusercontent.com/yoshiharuyamashita/blackburn/186f596907bd6ab0c746631bf8b66978b8072eef/layouts/partials/share.html
```

