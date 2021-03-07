#

## Usage

I currently use [hexo-images](https://github.com/sergeyzwezdin/hexo-images) to automatically create responsive versions for image used in blog posts.  It takes quite a bit of time to generate the versions of all images, so when deploying any commits to Git, be sure to disable `hexo-images` in the `_config.yml`.  Before deploying, make sure to build any new images that were added, e.g.

```bash
# enable hexo-images in the config
$ hexo generate
# commit all newly generated images
# then disable hexo-images in the config
``` 

## How to set up custom domain with this blog

See: https://stackoverflow.com/questions/49124018/gitlab-custom-domain-failed-to-verify-domain-ownership/49124195

