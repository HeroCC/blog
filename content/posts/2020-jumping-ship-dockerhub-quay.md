---
title: "Jumping Ship: Migrating your container images from DockerHub to Quay.io"
date: 2020-09-14T18:31:03+02:00
---

Here are the steps I took to move my images from DockerHub to Quay. If you're not interested in my thoughts or background, or know what you're doing, scroll down to "Moving the Images".

## Background

I host several repositories at DockerHub, both for work and personally.
When DockerHub announced they would be [deleting container images](https://www.docker.com/pricing/resource-consumption-updates) not used within 6 months, I understood. 
They provide a great service to the community, and frankly I'm surprised this didn't happen sooner. Docker Inc., and it's (new-ish) owner Mirantis, don't have the cash to burn like the Google and Amazon do. 
Still though, it's a bit sad, and can be frusterating. I believe DockerHub suffers from the same issue that [GitHub / Git](https://stackoverflow.com/q/13321556/1709894) does. For people not in-the-know, they are synonyms. This is perpetuated by the fact that the default docker registry for everyone using the Docker container implementation, is docker.io. 

If you weren't aware, whether you're new or never looked into the matter much, the command `docker pull herocc/my-cool-image:latest` actually maps to `docker pull docker.io/herocc/my-cool-image:latest`. See the difference there?
The default container registry for `docker`, is DockerHub. They look similar, but aren't the same.

Nonetheless, docker (the container engine) and DockerHub (the container registry) aren't the same. And now, lots of images (*4.5PB worth!!*) are slated for deletion. You have two options: pony up for a premium [DockerHub subscription](https://www.docker.com/pricing) for you or your company, or if you *really* need those oft-unused images to still be publicly avaliable with a $0 budget, you can move to a another registry, such as quay.io.

[Quay.io](https://quay.io) is another container registry, which as of this writing, offers a free public repo with unlimited retention. It is run by RedHat (and it's parent company, IBM), which has been making huge investments into containerization tech. I trust it will stick around for a while, and has a lot of nice features compared to DockerHub which I wasn't aware of before (though that's a point for another article). 

Anyway, on to the part you actually cared about: migrating the images. 

## Getting Ready

You're going to need an account on Quay to be able to push new images. Navigate to [quay](https://quay.io/repository/) and sign in or make an account. Then, click the "Create new repository" button. Fill this in as you like. For mine, I selected "Public", and "(Empty repository)". You can look into using Quay for your builds if you'd like, but that's out of scope for this post. 

Once you have your repo, you're going to need to sign in to quay locally to get permission to push image tags. Technically you can `docker login quay.io` with the username and password you just used, but for better security, you should follow these next steps. Click your username in the top right corner, click "Account Settings", and "Generate Encrypted Password". Once you get through the prompts, you'll find yourself with a long string of text. You can now run `docker login quay.io`, providing your standard username, and the encrypted password. 

You should see something along the lines of "Successful Login", and once you do, you're ready for the next step.

## Moving the Images

One way of moving the images is to pull them each down to your computer, `docker tag my-old/repo:tag quay.io/my-new/repo:tag`, but that is slow and inefficient. You need to manually pull each tag from the old registry, re-tag it, and push it back up again. If you have a lot of images, this could take a long time. Fortunately, there's an alternative: Skopeo.

Skopeo is a "command line utility that performs various operations on container images and image repositories". One of the features it provides is copying images from one registry to another -- either one-off, or syncing entire repos and registries. We will be using their `skopeo sync` feature to move over all of our tags.

First, lets install skopeo. This depends on your operating system, so you should read [their installation instructions](https://github.com/containers/skopeo/blob/master/install.md) and come back once you've finished. 

Once you finished installing skopeo, you should see something like this:
```
$ skopeo --version
skopeo version 1.1.1
```

If you do, that means you're ready to go! Make sure you're logged in to quay by running `skopeo login quay.io`, and you're ready to construct your command.

If your old image repo is hosted on DockerHub, it's full link is `docker.io/YOUR-ORG-OR-NAME/REPO-NAME`. Your repo on quay, the one you just made, is likely similar, `quay.io/YOUR-ORG-OR-NAME/REPO-NAME`. All you have to do is tell skopeo to sync one to the other.

If you have a lot of images this may take a while, so consider starting a `screen` session (something like `screen -S quaysync`). Then, replace the links below with your repo, and hit enter:

```bash
$ skopeo sync --src docker --dest docker docker.io/YOUR-ORG/REPO-NAME quay.io/YOUR-ORG

INFO[0000] Tag presence check              imagename=docker.io/org/repo tagged=false
INFO[0000] Getting tags                    image=docker.io/org/repo
INFO[0000] Copying image tag 1/2295        from="docker://org/repo:r8842" to="docker://quay.io/org/repo:r8842"
...
```

Skopeo will index all of the tags, and one by one copy the relevant blobs from DockerHub to Quay. You can refresh Quay to see this as it progresses.

And that's it! Once skopeo finishes, your tags will be safe over on quay. You should tell users to pull images from the new source, and make sure your CI system is set to push there instead (or at least schedule a cron job to sun `skopeo` again).

I hope this helped you in some way and that you enjoyed the read. Until next time, cya!

