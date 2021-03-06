Pull MODIS data via FTP
=======================

FTP access through Python
-------------------------

One way to pull data using ftp with Python is to use the module `ftplib <http://docs.python.org/library/ftplib.html>`_.

A typical task we might wish to do is to pull MODIS product data. Let us consider the `MODIS LAI/fAPAR product <https://lpdaac.usgs.gov/products/modis_products_table/leaf_area_index_fraction_of_photosynthetically_active_radiation>`_ MCD15A2. We can download the data from various sources (see e.g. the `USGS Land processes DAAC <https://lpdaac.usgs.gov/get_data>`_). One choice then might be `REVERB <http://reverb.echo.nasa.gov/reverb/#utf8=%E2%9C%93&spatial_map=satellite&spatial_type=rectangle>`_, where we can set an appropriate date range for our search and use suitable keywords (e.g. `MODIS LAI MCD15A2 <http://reverb.echo.nasa.gov/reverb/#utf8=%E2%9C%93&spatial_map=satellite&spatial_type=rectangle&keywords=MODIS%20LAI%20MCD15A2&temporal_start=2011-09-01%2000%3A00%3A00&temporal_end=2011-10-17%2023%3A59%3A59>`_). There are various ways of implementing a spatial search, but we can for example search by granule for tile `h09v05 <http://modis-land.gsfc.nasa.gov/MODLAND_grid.htm>`_. From this, we can learn that the files are available online, e.g. `ftp://e4ftl01.cr.usgs.gov/MODIS_Composites/MOTA/MCD15A2.005/2011.08.29/ <ftp://e4ftl01.cr.usgs.gov/MODIS_Composites/MOTA/MCD15A2.005/2011.08.29/>`_.


To connect to this site (anonymous login) from within Python:

::

    #!/usr/bin/env python
    
    import ftplib
    import numpy as np
    
    ftp = ftplib.FTP("e4ftl01.cr.usgs.gov")
    ftp.login("anonymous", "thisisme")
    ftp.cwd('MODIS_Composites/MOTA/MCD15A2.005')
    data = []
    ftp.dir(data.append)
    print data[:10]
    

::

    ['total 53600', 'drwxr-xr-x  2 90 122882 Jun 20  2008 2002.07.04', 'drwxr-xr-x  2 90 116736 Jun 20  2008 2002.07.12', 'drwxr-xr-x  2 90 118786 Jun 20  2008 2002.07.20', 'drwxr-xr-x  2 90 116736 Jun 20  2008 2002.07.28', 'drwxr-xr-x  2 90 118786 Jun 20  2008 2002.08.05', 'drwxr-xr-x  2 90 114688 Jun 20  2008 2002.08.13', 'drwxr-xr-x  2 90 118786 Jun 20  2008 2002.08.21', 'drwxr-xr-x  2 90 122880 Jun 20  2008 2002.08.29', 'drwxr-xr-x  2 90 116736 Jun 20  2008 2002.09.06']
    



We can see that the listing is the same form as we get from `ls -l <http://www2.geog.ucl.ac.uk/~plewis/geogg122/unix_intro.html#ls-l>`_
and that the variable data is a list containing strings. We note that the first element of the list is a header and so we should get data only from ``data[1:]``. We can retrieve the directory names with:

::

    dirnames = [i.split()[-1] for i in data[1:]]
    



We might further split this into year, month and day arrays and convert to ``int``:


>>> dirinfo = np.array([[ int(j) for j in i.split()[-1].split('.')] for i in data[1:]])




This operates on the fields ``data[1:]`` which are split (``i.split()``) then we take the final field (``i.split()[-1]``) which will usualy be something like ``2002.07.04``. This is then split on the separator ``.`` (``i.split()[-1].split('.')``) and converted to integer (``int(j)``).

So long as the directory names are of this pattern, this line of code will work, but sometimes there might be other files or directories that cause this to fail. What we are trying to achieve is an array of lists with 3 elements, representing the year, month and day as integers (e.g. ``[[2004,7,4],...]``). If, for example, ``data[1:]`` contains elements that are not of this pattern and e.g. fail when we try to convert them to integer, the code will fail at this point. 

There are several options in that case. We could just exit and tell the user that we have an unexpected pattern of directory names (but that is often rather frustrating for anyone trying to use the code) or we could attempt to foresee this problem and get around it. There are several ways this could be achieved, one is to check whether the string we are going to convert has all numerical characters:


>>> print unicode('1234').isnumeric() 
True



>>> print unicode('1234not').isnumeric()
False




in which case we can do e.g.:


>>> j = '1234'
>>> print (unicode(j).isnumeric() and int(j)) or 0
1234



>>> j = '1234not'
>>> print (unicode(j).isnumeric() and int(j)) or 0
0





so that we print some default value if ``j`` is not numerical. In fact, we want to return a triplet:


>>> i = 'hello there 1.2.3'
>>> [ (len(k) == 3 and k) or [0,0,0] for k in [[ (unicode(j).isnumeric() and int(j)) or 0 for j in i.split()[-1].split('.')] 




where we return ``int(j)`` if ``j`` is numerical, or 0 otherwise, then force the array to be of length 3, or return ``[0,0,0]``.

This should be quite robust now:

::

    dirinfo = np.array([ (len(k) == 3 and k) or [0,0,0] for k in [[ (unicode(j).isnumeric() and int(j)) or 0 for j in i.split()[-1].split('.')] for i in data[1:]]])
    print dirinfo
    

::

    [[2002    7    4]
     [2002    7   12]
     [2002    7   20]
     ..., 
     [2011    9   30]
     [2011   10    8]
     [2011   10   16]]
    



Now, suppose we want to access data for a particular month and year. The year information is stored in ``dirinfo[:,0]``, etc. so we can use ``np.where()`` to filter the array:

::

    thisMonth = 7
    thisYear = 2011
    yearMatch = dirinfo[:,0] == thisYear
    monthMatch = dirinfo[:,1] == thisMonth
    allMatch = yearMatch * monthMatch
    w = np.where(allMatch)[0]
    print w
    

::

    [414 415 416 417]
    



From which we learn that items ``414, 415, 416, 417`` are what we want, so the directories we want are:

::

    wantDirs = np.array(dirnames)[w]
    



Probably the most complicated part for newbies using ``ftplib`` is this next bit. This is just a 'callback' function that needs to be defined. It will be sent a block of data (``block``) and do something with it. Most typically, this will be just write it to a file, so here we assume ``file`` is an open file:

::

    def handleDownload(block):
        file.write(block)
    



So we can now change directory to each and find the tile:

::

    thisTile = 'h09v05'
    for i in wantDirs:
        ftp.cwd(i)
        this = []
        ftp.dir(this.append)
        filenames = [i.split()[-1] for i in this[1:]]
        fileinfo = np.array([i.split()[-1].split('.') for i in this[1:]])
        tileMatch = np.array([i[2] == thisTile for i in fileinfo])
        w = np.where(tileMatch)[0]
        for f in np.array(filenames)[w]:
            print 'Opening local file ' + f
            file = open(f, 'wb')
    	print 'Getting ' + f
            ftp.retrbinary('RETR %s'%f,handleDownload)
        ftp.cwd('..')
    
    ftp.quit()
    

::

    Opening local file MCD15A2.A2011185.h09v05.005.2011213154534.hdf
    Getting MCD15A2.A2011185.h09v05.005.2011213154534.hdf
    Opening local file MCD15A2.A2011185.h09v05.005.2011213154534.hdf.xml
    Getting MCD15A2.A2011185.h09v05.005.2011213154534.hdf.xml
    Opening local file MCD15A2.A2011193.h09v05.005.2011203175347.hdf
    Getting MCD15A2.A2011193.h09v05.005.2011203175347.hdf
    Opening local file MCD15A2.A2011193.h09v05.005.2011203175347.hdf.xml
    Getting MCD15A2.A2011193.h09v05.005.2011203175347.hdf.xml
    Opening local file MCD15A2.A2011201.h09v05.005.2011210161955.hdf
    Getting MCD15A2.A2011201.h09v05.005.2011210161955.hdf
    Opening local file MCD15A2.A2011201.h09v05.005.2011210161955.hdf.xml
    Getting MCD15A2.A2011201.h09v05.005.2011210161955.hdf.xml
    Opening local file MCD15A2.A2011209.h09v05.005.2011220122803.hdf
    Getting MCD15A2.A2011209.h09v05.005.2011220122803.hdf
    Opening local file MCD15A2.A2011209.h09v05.005.2011220122803.hdf.xml
    Getting MCD15A2.A2011209.h09v05.005.2011220122803.hdf.xml
    



Here, we loop over ``wantDirs``, change directory into the required directory, get a list of all filenames and see which ones match the tile we want, then loop over these files and download (using the function ``handleDownload()`` to do the file saving). Finally, we change directory back up one level, and quit from the ftp session once we have everything we want.


Exercise
--------

You can put all of these lines of code together into something more coherent. An example is `given here <python/modisftpBroken.py>`_ , but it has two bugs. Try to spot and fix the bugs. You might find using `pdb` useful for this. To verify your bug fixes, you can find a `fixed version of the code here <python/modisftp.py>`_.

Exercise
--------

Modify this code so that it allows you to download the MODIS snow products ``MYD10A2`` and ``MOD10A2`` and use this to download data for the tile ``h09v05`` for the years 2005.

**Hint** you will not find these data on the same ftp server.

You could also try to make the code a little more robust to errors. At present it does minimal checking, and there may be circumstances that it will fail.

You can find some code that will do the minimal part of the exercise `here <python/modisSnowFtp.py>`_, though you should not look at that except to check your own code.

More flexible code
------------------

A 'better' and slightly more flexible piece of code can be found `here <python/modisAlbedoftp.py>`_ that is set up to pull MODIS albedo data by default. This code is quite a bit more complicated than the previous example, partly because it has a command line parser in it. Try to follow the logic of this code, and add a parser to your own code.

Also of interest, `Jose <https://github.com/jgomezdans/modisficator/>`_ has made a more generic tool available for organising and accessing MODIS products.
