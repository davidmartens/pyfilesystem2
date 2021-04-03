Guide
=====

PyFilesytem provides an interface that simplifies most aspects of working with files and directories by providing a series of filesystem objects ("FS objects"). This guide covers what you need to know about working with FS objects.

Why use PyFilesystem?
~~~~~~~~~~~~~~~~~~~~~

If you are comfortable using the Python standard library, you may wonder, *why learn another API for working with files?*

PyFilesystem provides a :ref:`interface` that is generally simpler to understand and use than Python's standard ``os`` and ``io`` modules.  PyFilesystem's :ref:`interface` contains fewer edge cases, and fewer ways to shoot yourself in the foot, relative to Python's standard ``os`` and ``io`` modules.  PyFilesystem provides a simplicity and consistency that alone may justify its use, but other compelling reasons exist for using PyFilesystem even for straightforward filesystem functionalities.

FS objects provide an abstraction that conveniently allows code expressions that are agnostic to the physical location of files. For example, if a function was written for searching a directory for duplicates files, this function would work unaltered operating on a directory within your hard-drive, a directory within a zip file, a directory on a FTP server, a directory on Amazon S3, etc.  If a FS object is available for your chosen filesystem (or any data store that resembles a filesystem), PyFilesystem's consistent API allows you to easily access files on your chosen filesystem. 

FS objects allow deferring a decision how to store your files (e.g., disk filesystem, FTP server, etc.) and allow easily moving your files in the future. Only a single-line change should be needed for PyFilesystem to support changing how your files are stored.

PyFilesystem is also platform agnostic, so your code will work without modification on Linux, MacOS, and Windows operating systems.  In this way, PyFilesystem extends platform-agnosticism features within Python's standard modules.

PyFilesystem's FS objects also provides unit-testing benefits.  For example, unit tests can modify code for accessing files within a traditional disk filesystem with code for accessing files within an in-memory filesystem.  This capability allows for writing unit tests without having to manage (or mock) file I/O. 

Opening Filesystems
~~~~~~~~~~~~~~~~~~~

PyFilesystem provides two ways for opening a filesystem.  A discussion of each way for opening a filesystem, with examples, follows.

    Importing a Filesystem Class
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The first and most natural way to open a filesystem is to import and instantiate an appropriate filesystem class.

An example of importing and instantiating a :class:`~fs.osfs.OSFS` (Operating System File System) class is shown below.  The OSFS filesystem provides access to files and directories on a physical hard drive::

    >>> from fs.osfs import OSFS
    >>> home_fs = OSFS("~/")

The logic shown above constructs a OSFS object to manage files and directories under a provided system path. In this example, the provided system path is ``'~/'``, which reflects a shortcut for your home directory within PyFilesystem.

The example shown below extends the example shown above to list files and directories at the top level within the provided system path::

    >>> home_fs.listdir('/')
    ['world domination.doc', 'paella-recipe.txt', 'jokes.txt', 'projects']

Notice that the argument to ``listdir`` is a single forward slash, indicating that we want to list the *root* of the filesystem.  Recall also that the provided system path specified the filesystem as ``'~/'``.

PyFilesystem's platform agnosticism provides for a forward slash as a path delimiter (even on Windows). See :ref:`paths` for details.

Some filesystem interfaces may require additional or different constructor arguments. For example, instantiating a FTP filesystem requires a URL at which the root filesystem is stored::

    >>> from ftpfs import FTPFS
    >>> debian_fs = FTPFS('ftp.mirror.nl')
    >>> debian_fs.listdir('/')
    ['debian-archive', 'debian-backports', 'debian', 'pub', 'robots.txt']

    URL-like Opener Expressions
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~
    
A second, and more general way of opening filesystems objects, is through an *opener* expression which identifies a filesystem through a URL-like syntax. An example of opening a user's home directory, through an opener expression, is shown below::

    >>> from fs import open_fs
    >>> home_fs = open_fs('osfs://~/')
    >>> home_fs.listdir('/')
    ['world domination.doc', 'paella-recipe.txt', 'jokes.txt', 'projects']

As seen above, the opener expression specifies a filesystem class through a scheme specifier.  One benefit of using a scheme specifier when opening filesystems is that the scheme specifier could be expressed through a traditional Python format expression, allowing the filesystem argument to be determined at runtime, possibly through configuration data read from a file.

If no scheme is specified within the argument to open_fs, PyFilesystem creates an OSFS instance at the current working directory. So the following example reflects an equivalent way of opening your home directory::

    >>> from fs import open_fs
    >>> home_fs = open_fs('.')
    >>> home_fs.listdir('/')
    ['world domination.doc', 'paella-recipe.txt', 'jokes.txt', 'projects']

Tree Printing
~~~~~~~~~~~~~

Developers often want a function for creating a simple illustration of the contents of a filesystem.  PyFilesystem provides an easy mechanism for displaying filesystem contents.

Calling :meth:`~fs.base.FS.tree` on a FS object will print an ascii tree view of your filesystem. Here's an example::

    >>> from fs import open_fs
    >>> my_fs = open_fs('.')
    >>> my_fs.tree()
    ├── locale
    │   └── readme.txt
    ├── logic
    │   ├── content.xml
    │   ├── data.xml
    │   ├── mountpoints.xml
    │   └── readme.txt
    ├── lib.ini
    └── readme.txt

This can be a useful debugging aid!


Closing Filesystem Objects
~~~~~~~~~~~~~~~~~~~~~~~~~~

FS objects have a :meth:`~fs.base.FS.close` method which performs any required clean-up actions. For many filesystems (notably :class:`~fs.osfs.OSFS`), the ``close`` method does very little. Other filesystems may only finalize files or release resources once ``close()`` is called.

Code can call ``close`` explicitly once the code is finished using a filesystem. For example::

    >>> home_fs = open_fs('osfs://~/')
    >>> home_fs.writetext('reminder.txt', 'buy coffee')
    >>> home_fs.close()

If you use FS objects as a context manager, ``close`` will be called automatically. The following example is equivalent to the previous example::

    >>> with open_fs('osfs://~/') as home_fs:
    ...    home_fs.writetext('reminder.txt', 'buy coffee')

Using FS objects as a context manager is recommended as it will ensure every FS instance is properly closed.

Directory Contents
~~~~~~~~~~~~~~~~~~

Filesystem objects have a :meth:`~fs.base.FS.listdir` method which is similar to ``os.listdir``; this method receives as an argument a path to a directory and returns a list of file names. Here's an example::

    >>> home_fs.listdir('/projects')
    ['fs', 'moya', 'README.md']

An alternative method exists for listing directories; :meth:`~fs.base.FS.scandir` returns an *iterable* of :ref:`info` objects. Here's an example::

    >>> directory = list(home_fs.scandir('/projects'))
    >>> directory
    [<dir 'fs'>, <dir 'moya'>, <file 'README.md'>]

Info objects have a number of advantages over just a filename. For instance you can tell if an info object references a file or a directory with the :attr:`~fs.info.Info.is_dir` attribute, without an additional system call. Info objects may also contain information such as size, modified time, etc. if you request it in the ``namespaces`` parameter.


.. note::

    The reason that ``scandir`` returns an iterable rather than a list, is that it can be more efficient to retrieve directory information in chunks if the directory is very large, or if the information must be retrieved over a network.

Additionally, FS objects have a :meth:`~fs.base.FS.filterdir` method which extends ``scandir`` with the ability to filter directory contents by wildcard(s). Here's how you might find all the Python files in a directory:

    >>> code_fs = OSFS('~/projects/src')
    >>> directory = list(code_fs.filterdir('/', files=['*.py']))

By default, the resource information objects returned by ``scandir`` and ``listdir`` will contain only the file name and the ``is_dir`` flag. You can request additional information with the ``namespaces`` parameter. Here's how you can request additional details (such as file size and file modified times)::

    >>> directory = code_fs.filterdir('/', files=['*.py'], namespaces=['details'])

This will add a ``size`` and ``modified`` property (and others) to the resource info objects. Which makes code such as this work::

    >>> sum(info.size for info in directory)

See :ref:`info` for more information.

Sub Directories
~~~~~~~~~~~~~~~

PyFilesystem has no notion of a *current working directory*, so you won't find a ``chdir`` method on FS objects. Fortunately you won't miss it; working with sub-directories is a breeze with PyFilesystem.

You can always specify a directory with methods which accept a path. For instance, ``home_fs.listdir('/projects')`` would get the directory listing for the `projects` directory. Alternatively, you can call :meth:`~fs.base.FS.opendir` which returns a new FS object for the sub-directory.

For example, here's how you could list the directory contents of a `projects` folder in your home directory::


    >>> home_fs = open_fs('~/')
    >>> projects_fs = home_fs.opendir('/projects')
    >>> projects_fs.listdir('/')
    ['fs', 'moya', 'README.md']

When you call ``opendir``, the FS object returns an instance of a :class:`~fs.subfs.SubFS`. If you call any of the methods on a ``SubFS`` object, it will be as though you called the same method on the parent filesystem with a path relative to the sub-directory.

The :class:`~fs.base.FS.makedir` and :class:`~fs.base.FS.makedirs` methods also return ``SubFS`` objects for the newly create directory. Here's how you might create a new directory in ``~/projects`` and initialize it with a couple of files::

    >>> home_fs = open_fs('~/')
    >>> game_fs = home_fs.makedirs('projects/game')
    >>> game_fs.touch('__init__.py')
    >>> game_fs.writetext('README.md', "Tetris clone")
    >>> game_fs.listdir('/')
    ['__init__.py', 'README.md']

Working with ``SubFS`` objects means that you can generally avoid writing much path manipulation code, which tends to be error prone.

Working with Files
~~~~~~~~~~~~~~~~~~

You can open a file from a FS object with :meth:`~fs.base.FS.open`, which is very similar to ``io.open`` in the standard library. Here's how you might open a file called "reminder.txt" in your home directory::

    >>> with open_fs('~/') as home_fs:
    ...     with home_fs.open('reminder.txt') as reminder_file:
    ...        print(reminder_file.read())
    buy coffee

In the case of a ``OSFS``, a standard file-like object will be returned. Other filesystems may return a different object supporting the same methods. For instance, :class:`~fs.memoryfs.MemoryFS` will return a ``io.BytesIO`` object.

PyFilesystem also offers a number of shortcuts for common file related operations. For instance, :meth:`~fs.base.FS.readbytes` will return the file contents as a bytes, and :meth:`~fs.base.FS.readtext` will read unicode text. These methods is generally preferable to explicitly opening files, as the FS object may have an optimized implementation.

Other *shortcut* methods are :meth:`~fs.base.FS.download`, :meth:`~fs.base.FS.upload`, :meth:`~fs.base.FS.writebytes`, :meth:`~fs.base.FS.writetext`.

Walking
~~~~~~~

Often you will need to scan the files in a given directory, and any sub-directories. This is known as *walking* the filesystem.

Here's how you would print the paths to all your Python files in your home directory::

    >>> from fs import open_fs
    >>> home_fs = open_fs('~/')
    >>> for path in home_fs.walk.files(filter=['*.py']):
    ...     print(path)

The ``walk`` attribute on FS objects is instance of a :class:`~fs.walk.BoundWalker`, which should be able to handle most directory walking requirements.

See :ref:`walking` for more information on walking directories.

Globbing
~~~~~~~~

Closely related to walking a filesystem is *globbing*, which is a slightly higher level way of scanning filesystems. Paths can be filtered by a *glob* pattern, which is similar to a wildcard (such as ``*.py``), but can match multiple levels of a directory structure.

Here's an example of globbing, which removes all the ``.pyc`` files in your project directory::

    >>> from fs import open_fs
    >>> open_fs('~/project').glob('**/*.pyc').remove()
    62

See :ref:`globbing` for more information.


Moving and Copying
~~~~~~~~~~~~~~~~~~

You can move and copy file contents with :meth:`~fs.base.FS.move` and :meth:`~fs.base.FS.copy` methods, and the equivalent :meth:`~fs.base.FS.movedir` and :meth:`~fs.base.FS.copydir` methods which operate on directories rather than files.

These move and copy methods are optimized where possible, and depending on the implementation, they may be more performant than reading and writing files.

To move and/or copy files *between* filesystems (as apposed to within the same filesystem), use the :mod:`~fs.move` and :mod:`~fs.copy` modules. The methods in these modules accept both FS objects and FS URLS. For instance, the following will compress the contents of your projects folder::

    >>> from fs.copy import copy_fs
    >>> copy_fs('~/projects', 'zip://projects.zip')

Which is the equivalent to this, more verbose, code::

    >>> from fs.copy import copy_fs
    >>> from fs.osfs import OSFS
    >>> from fs.zipfs import ZipFS
    >>> copy_fs(OSFS('~/projects'), ZipFS('projects.zip'))

The :func:`~fs.copy.copy_fs` and :func:`~fs.copy.copy_dir` functions also accept a :class:`~fs.walk.Walker` parameter, which can you use to filter the files that will be copied. For instance, if you only wanted back up your python files, you could use something like this::

    >>> from fs.copy import copy_fs
    >>> from fs.walk import Walker
    >>> copy_fs('~/projects', 'zip://projects.zip', walker=Walker(filter=['*.py']))

An alternative to copying is *mirroring*, which will copy a filesystem them keep it up to date by copying only changed files / directories. See :func:`~fs.mirror.mirror`.
