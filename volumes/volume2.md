# L08-03 Volume 2 — What Actually Happened

## What the guide expected vs. what happened

The original guide assumes `nano` and `vi` are available inside the nginx container.
In practice, the **nginx:latest image is minimal** — it ships with neither editor,
and `apt-get update` failed with **403 Forbidden** errors because outbound HTTP
to `deb.debian.org` was blocked by the network. This meant no packages could be installed.

## Workaround — write a file without an editor

Instead of nano, we used `echo` to write directly to the file:

    echo "hello volume!" > test.txt

Verify it was written:

    cat test.txt

## The volume still worked perfectly

Even though we couldn't use nano, the core lesson was confirmed:

1. Created the volume and ran the container

        docker volume create myvol
        docker run -d --name voltest -v myvol:/app nginx:latest
        docker exec -it voltest bash

2. Wrote a file into the volume-mounted folder (`/app`)

        cd app
        echo "hello volume!" > test.txt
        exit

3. Stopped and removed the container — volume data is NOT deleted

        docker stop voltest
        docker rm voltest

4. Ran a brand-new container with the same volume — file was still there

        docker run -d --name voltest -v myvol:/app nginx:latest
        docker exec -it voltest bash
        cd app
        cat test.txt
        # Output: hello volume!

## Key mistake to avoid

`docker rm` removes a **container**, not a volume.
To remove a volume you must use `docker volume rm`:

    # Wrong — tries to remove a volume as if it were a container
    docker rm myvol

    # Correct
    docker volume rm myvol

Also, a volume cannot be removed while a container is still using it:

    docker stop voltest
    docker rm voltest
    docker volume rm myvol   # now it works

## Cleanup

    docker stop voltest
    docker rm voltest
    docker volume rm myvol
