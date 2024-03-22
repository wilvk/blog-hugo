# my blog repo

## for me:

to create a new post:

hugo new posts/extending-fibonacci.md

to create a local server showing drafts:

hugo -D server

to build and upload:

./build

images go in ./static/img/ and land on http://me.wvk.au/img/

## Changing domain name:

  find . -type f -name '*.html' -exec sed -i '' 's/https:\/\/blog.seso.io/http:\/\/me.wvk.au/g' {} +
  find . -type f -name '*.md' -exec sed -i '' 's/https:\/\/blog.seso.io/http:\/\/me.wvk.au/g' {} +

## check name has changed:

  grep -rn seso
