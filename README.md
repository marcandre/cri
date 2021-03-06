# Cri

[![Gem](http://img.shields.io/gem/v/cri.svg)](http://rubygems.org/gems/cri)
[![Travis](http://img.shields.io/travis/ddfreyne/cri.svg)](https://travis-ci.org/ddfreyne/cri)
[![Coveralls](http://img.shields.io/coveralls/ddfreyne/cri.svg)](https://coveralls.io/r/ddfreyne/cri)
[![Codeclimate](http://img.shields.io/codeclimate/github/ddfreyne/cri.svg)](https://codeclimate.com/github/ddfreyne/cri)
[![Inch](http://inch-ci.org/github/ddfreyne/cri.svg)](http://inch-ci.org/github/ddfreyne/cri/)

Cri is a library for building easy-to-use command-line tools with support for
nested commands.

## Requirements

Cri requires Ruby 2.3 or newer.

## Usage

The central concept in Cri is the _command_, which has option definitions as
well as code for actually executing itself. In Cri, the command-line tool
itself is a command as well.

Here’s a sample command definition:

```ruby
command = Cri::Command.define do
  name        'dostuff'
  usage       'dostuff [options]'
  aliases     :ds, :stuff
  summary     'does stuff'
  description 'This command does a lot of stuff. I really mean a lot.'

  flag   :h,  :help,  'show help for this command' do |value, cmd|
    puts cmd.help
    exit 0
  end
  flag   nil, :more,  'do even more stuff'
  option :s,  :stuff, 'specify stuff to do', argument: :required

  run do |opts, args, cmd|
    stuff = opts.fetch(:stuff, 'generic stuff')
    puts "Doing #{stuff}!"

    if opts[:more]
      puts 'Doing it even more!'
    end
  end
end
```

To run this command, invoke the `#run` method with the raw arguments. For
example, for a root command (the command-line tool itself), the command could
be called like this:

```ruby
command.run(ARGV)
```

Each command has automatically generated help. This help can be printed using
`Cri::Command#help`; something like this will be shown:

```
usage: dostuff [options]

does stuff

    This command does a lot of stuff. I really mean a lot.

options:

    -h --help      show help for this command
       --more      do even more stuff
    -s --stuff     specify stuff to do
```

### General command metadata

Let’s disect the command definition and start with the first five lines:

```ruby
name        'dostuff'
usage       'dostuff [options]'
aliases     :ds, :stuff
summary     'does stuff'
description 'This command does a lot of stuff. I really mean a lot.'
```

These lines of the command definition specify the name of the command (or the
command-line tool, if the command is the root command), the usage, a list of
aliases that can be used to call this command, a one-line summary and a (long)
description. The usage should not include a “usage:” prefix nor the name of
the supercommand, because the latter will be automatically prepended.

Aliases don’t make sense for root commands, but for subcommands they do.

### Command-line options

The next few lines contain the command’s option definitions:

```ruby
flag   :h,  :help,  'show help for this command' do |value, cmd|
  puts cmd.help
  exit 0
end
flag   nil, :more,  'do even more stuff'
option :s,  :stuff, 'specify stuff to do', argument: :required
```

Options can be defined using the following methods:

* `Cri::CommandDSL#option` or `Cri::CommandDSL#opt` (include an `argument` parameter: `:required` or `:optional` that specifies if the option requires or allows an argument)
* `Cri::CommandDSL#flag` (implies no arguments passed to option)

The following _deprecated_ methods can also be used to define options:

* `Cri::CommandDSL#required` (implies an option that requires an argument -- deprecated because `#required` suggests that the option is required, wich is incorrect; the _argument_ is required.)
* `Cri::CommandDSL#optional` (implies an option that can optionally include an argument -- deprecated because `#optional` looks too similar to `#option`.)

All these methods take these arguments:

1. a short option name
2. a long option name
3. a description
4. optional extra parameters

Either the short or the long form can be nil, but not both (because that
would not make any sense). In the example above, the `--more` option has no
short form.

Each of the above methods also take a block, which will be executed when the
option is found. The arguments to the block are the option value (`true` in
case the option does not have an argument) and the command.

#### Transforming options

The `:transform` parameter specifies how the value should be transformed. It takes any object that responds to `#call`:

```ruby
option :p, :port, 'set port', argument: :required,
  transform: -> (x) { Integer(x) }
```

The following example uses `#Integer` to transform a string into an integer:

```ruby
option :p, :port, 'set port', argument: :required, transform: method(:Integer)
```

The following example uses a custom object to perform transformation, as well as validation:

```ruby
class PortTransformer
  def call(str)
    raise ArgumentError unless str.is_a?(String)
    Integer(str).tap do |int|
      raise unless (0x0001..0xffff).include?(int)
    end
  end
end

option :p, :port, 'set port', argument: :required, transform: PortTransformer.new
```

Default values are not transformed:

```ruby
option :p, :port, 'set port', argument: :required, default: 8080, transform: PortTransformer.new
```

#### Options with default values

The `:default` parameter sets the option value that will be used if the option is passed without an argument or isn't passed at all:

```ruby
option :a, :animal, 'add animal', default: 'giraffe', argument: :optional
```

In the example above, the value for the `--animal` option will be the string
`"giraffe"`, unless otherwise specified:

```
OPTIONS
    -a --animal[=<value>]      add animal (default: giraffe)
```

#### Multivalued options

Each of these four methods take a `:multiple` parameter. When set to true, multiple
option valus are accepted, and the option values will be stored in an array.

For example, to parse the command line options string `-o foo.txt -o bar.txt`
into an array, so that `options[:output]` contains `[ 'foo.txt', 'bar.txt' ]`,
you can use an option definition like this:

```ruby
option :o, :output, 'specify output paths', argument: :required, multiple: true
```

This can also be used for flags (options without arguments). In this case, the
length of the options array is relevant.

For example, you can allow setting the verbosity level using `-v -v -v`. The
value of `options[:verbose].size` would then be the verbosity level (three in
this example). The option definition would then look like this:

```ruby
flag :v, :verbose, 'be verbose (use up to three times)', multiple: true
```

#### Skipping option parsing

If you want to skip option parsing for your command or subcommand, you can add
the `skip_option_parsing` method to your command definition and everything on your
command line after the command name will be passed to your command as arguments.

```ruby
command = Cri::Command.define do
  name        'dostuff'
  usage       'dostuff [args]'
  aliases     :ds, :stuff
  summary     'does stuff'
  description 'This command does a lot of stuff, but not option parsing.'

  skip_option_parsing

  run do |opts, args, cmd|
    puts args.inspect
  end
end
```

When executing this command with `dostuff --some=value -f yes`, the `opts` hash
that is passed to your `run` block will be empty and the `args` array will be
`["--some=value", "-f", "yes"]`.

### Argument parsing

Cri also supports parsing arguments, outside of options. To define the
parameters of a command, use `#param`, which takes a symbol containing the name
of the parameter. For example:

```ruby
command = Cri::Command.define do
  name        'publish'
  usage       'publish filename'
  summary     'publishes the given file'
  description 'This command does a lot of stuff, but not option parsing.'

  flag :q, :quick, 'publish quicker'
  param :filename

  run do |opts, args, cmd|
    puts "Publishing #{args[:filename]}…"
  end
end
```

The command in this example has one parameter named `filename`. This means that
the command takes a single argument, named `filename`.

If no parameters are specified, Cri performs no argument parsing or validation; any number of arguments is allowed. To explicitly specify that a command has no parameters, use `#no_params`:

```ruby
name        'reset'
usage       'reset'
summary     'resets the site'
description '…'
no_params

run do |opts, args, cmd|
  puts "Resetting…"
end
```

```
% my-tool reset x
reset: incorrect number of arguments given: expected 0, but got 1
% my-tool reset
Resetting…
%
```

A future version of Cri will likely make `#no_params` the default behavior.

As with options, parameter definitions take `transform:`, which can be used for transforming and validating arguments:

```ruby
param :port, transform: method(:Integer)
```

(*Why the distinction between argument and parameter?* A parameter is a name, e.g. `filename`, while an argument is a value for a parameter, e.g. `kitten.jpg`.)

### The run block

The last part of the command defines the execution itself:

```ruby
run do |opts, args, cmd|
  stuff = opts.fetch(:stuff, 'generic stuff')
  puts "Doing #{stuff}!"

  if opts[:more]
    puts 'Doing it even more!'
  end
end
```

The +Cri::CommandDSL#run+ method takes a block with the actual code to
execute. This block takes three arguments: the options, any arguments passed
to the command, and the command itself.

Instead of defining a run block, it is possible to declare a class, the
_command runner_ class (`Cri::CommandRunner`) that will perform the actual
execution of the command. This makes it easier to break up large run blocks
into manageable pieces.

### Subcommands

Commands can have subcommands. For example, the `git` command-line tool would be
represented by a command that has subcommands named `commit`, `add`, and so on.
Commands with subcommands do not use a run block; execution will always be
dispatched to a subcommand (or none, if no subcommand is found).

To add a command as a subcommand to another command, use the
`Cri::Command#add_command` method, like this:

```ruby
root_cmd.add_command(cmd_add)
root_cmd.add_command(cmd_commit)
root_cmd.add_command(cmd_init)
```

You can also define a subcommand on the fly without creating a class first
using `Cri::Command#define_command` (name can be skipped if you set it inside
the block instead):

```ruby
root_cmd.define_command('add') do
  # option ...
  run do |opts, args, cmd|
    # ...
  end
end
```

You can specify a default subcommand. This subcommand will be executed when the
command has subcommands, and no subcommands are otherwise explicitly specified:

```ruby
default_subcommand 'compile'
```

### Loading commands from separate files

You can use `Cri::Command.load_file` to load a command from a file.

For example, given the file _commands/check.rb_ with the following contents:

```ruby
name        'check'
usage       'check'
summary     'runs all checks'
description '…'

run do |opts, args, cmd|
  puts "Running checks…"
end
```

To load this command:

```ruby
Cri::Command.load_file('commands/check.rb')
```

`Cri::Command.load_file` expects the file to be in UTF-8.

You can also use it to load subcommands:

```ruby
root_cmd = Cri::Command.load_file('commands/nanoc.rb')
root_cmd.add_command(Cri::Command.load_file('commands/comile.rb'))
root_cmd.add_command(Cri::Command.load_file('commands/view.rb'))
root_cmd.add_command(Cri::Command.load_file('commands/check.rb'))
```

#### Automatically inferring command names

Pass `infer_name: true` to `Cri::Command.load_file` to use the file basename as the name of the command.

For example, given a file _commands/check.rb_ with the following contents:

```ruby
usage       'check'
summary     'runs all checks'
description '…'

run do |opts, args, cmd|
  puts "Running checks…"
end
```

To load this command and infer the name:

```ruby
cmd = Cri::Command.load_file('commands/check.rb', infer_name: true)
```

`cmd.name` will be `check`, derived from the filename.

## Contributors

* Bart Mesuere
* Ken Coar
* Tim Sharpe
* Toon Willems

Thanks for Lee “injekt” Jarvis for [Slop](https://github.com/injekt/slop),
which has inspired the design of Cri 2.0.
