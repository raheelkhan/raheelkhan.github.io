---
title: Python argparse and subparsers
date: 2017-09-30
categories:
---

Often times we have to create command line utilities and it doesn’t make sense every time to put one command in a separate file specially when you have a bunch of related functionalities.

This is a quick tutorial for handling sub commands in python which shows how we can write multiple functions in one module and call them via terminal.

I will create a simple `indices.py` file which will contain two commands

1. create
2. delete

Python provides us with `argsparse` module to handle command line arguments.

> The argparse module makes it easy to write user-friendly command-line interfaces. The program defines what arguments it requires, and argparse will figure out how to parse those out of `sys.argv`. The argparse module also automatically generates help and usage messages and issues errors when users give the program invalid arguments.

Lets create a file called `indices.py` and add two functions

```python
def create(name):
    """

    :param name: Name of index
    :return: None
    """
    print "Creating index with name: %s" % name

def delete(name):
    """

    :param name: Name of index
    :return: None
    """
    print "Deleting index with name: %s" % name
```

Now we will create an `ArgumentParser` object which will serve as a file level parser

```python
if __name__ == '__main__':
    parser = ArgumentParser(description='Index related commands', usage='''
        $ python indices.py command

        List of commands:
            create      Create index on database
            delete      Delete index from database

    ''')
```

At this point we will be able to see the usage instructions of our `indices.py` file

```
root@583f3dc0bde4:/backend# python indices.py -h
usage:
 $ python -m src.commands.indices command [<args>]
List of commands:
    Create index on database
    Delete index from database

Index related commands

positional arguments:
    {}

optional arguments:
    -h, --help show this help message and exit
```

## Adding subparsers

Python’s argsparse module lets you add subparsers for your different sub-commands within the same module. In our case its create and delete commands

```python
subparsers = parser.add_subparsers()

parser_create = subparsers.add_parser('create', help='creates a new index', usage='''
    $ python indices.py create [<args>]
''')
parser_create.add_argument('index', type=str, help='name of index')
parser_create.set_defaults(function=create)

parser_delete = subparsers.add_parser('delete', help='delete index', usage='''
    $ python indices.py delete [<args>]
''')
parser_delete.add_argument('index', type=str, help='name of index')
parser_delete.set_defaults(function=delete)
```

> ArgumentParser supports the creation of such sub-commands with the `add_subparsers()` method. The `add_subparsers()` method is normally called with no arguments and returns a special action object. This object has a single method, `add_parser()`, which takes a command name and any ArgumentParser constructor arguments, and returns an ArgumentParser object that can be modified as usual.

What we have done above is that we created two ArgumentParser objects for our sub commands. For the sake of convention we use variable names as parser_command.

Next we’ve defined our positional argument named index

The `set_defaults` method is what we are interested in most. This method takes a keyword argument called function and let you assign the name of a command to execute which it encounters in the `sys.argv`

Now if you see the -h of your file you will see the complete out of the usage something like

```
root@583f3dc0bde4:/backend# python -m src.commands.indices -h
usage:
$ python -m src.commands.indices <command> [<args>]

List of commands:
create create index on database
delete delete index from database

Index related commands

positional arguments:
{create,delete}
create     creates a new index
delete     delete index

optional arguments:
-h, --help show this help message and exit
```

Finally we need to enable our python code to call our commands with their arguments. In this case the `index`

```python
arguments = parser.parse_args()
arguments.function(arguments.index)
```

At this point you can call methods create and delete as follows

```
root@583f3dc0bde4:/backend# indices.py create catalog

root@583f3dc0bde4:/backend# indices.py delete catalog
```
