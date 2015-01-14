smoynes online blog
===================

Powered by [Pelican](http://blog.getpelican.com/).

how to blog by smoynes
----------------------

Create a virtualenv and install requirements:

    $ virtualenv env
    $ source env/bin/activate
    $ pip install -r requirements.txt

Start the development server and open the blog:

    $ make devserver
    $ open http://localhost:8000

Build the site and publish it:

    $ make html
    $ make rsync_upload SSH_USER=user SSH_HOST=hostname
