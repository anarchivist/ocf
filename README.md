# OCF site

this is my site on [OCF](https://ocf.berkeley.edu), the Open Computing Facility at UC Berkeley. it is built using [Hugo](https://gohugo.io/) using my fork of [hugo-tufte](https://github.com/loikein/hugo-tufte).

building and deployment happens through the magic of Github Actions, which runs a build workflow and then rsyncs the files to OCF.

OCF side is locked down through ssh `authorized_keys` magic and a specific deploy key.