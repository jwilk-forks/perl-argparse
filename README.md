NAME
    Getopt::ArgParse - Parsing command line arguments with a richer and more
    user-friendly API interface, similar to python's argparse but with
    perlish extras.

    In particular, the modules provides the following features:

      - generating usage messages
      - storing parsed arg values in an object, which can be also used to
        load configuration values from files and therefore the ability for
        applications to combine configurations in a single interface
      - A more user-friendly interface to specify arguments, such as
        argument types, argument values split, etc.
      - Subcommand parsing, such svn <command>
      - Supporting both flag based named arguments and positional arguments

VERSION
    version 1.0.3

SYNOPSIS
     use Getopt::ArgParse;

     $ap = Getopt::ArgParse->new_parser(
            prog        => 'MyProgramName',
            description => 'This is a program',
            epilog      => 'This appears at the bottom of usage',
     );

     # Parse an option: '--foo value' or '-f value'
     $ap->add_arg('--foo', '-f', required => 1);

     # Parse a boolean: '--bool' or '-b' using a different name from
     # the option
     $ap->add_arg('--bool', '-b', type => 'Bool', dest => 'boo');

     # Parse a positional option.
     # But in this case, better using subcommand. See below
     $ap->add_arg('command', required => 1);

     # $ns is also accessible via $ap->namespace
     $ns = $ap->parse_args(split(' ', 'test -f 1 -b'));

     say $ns->command; # 'test'
     say $ns->foo;     # false
     say $ns->boo;     # false
     say $ns->no_boo;   # true - 'no_' is added for boolean options

     # You can continue to add arguments and parse them again
     # $ap->namespace is accumulatively populated

     # Parse an Array type option and split the value into an array of values
     $ap->add_arg('--emails', type => 'Array', split => ',');
     $ns = $ap->parse_args(split(' ', '--emails a@perl.org,b@perl.org,c@perl.org'));
     # Because this is an array option, this also allows you to specify the
     # option multiple times and splitting
     $ns = $ap->parse_args(split(' ', '--emails a@perl.org,b@perl.org --emails c@perl.org'));

     # Below will print: a@perl.org|b@perl.org|c@perl.org|a@perl.org|b@perl.org|c@perl.org
     # Because Array types are appended
     say join('|', $ns->emails);

     # Parse an option as key,value pairs
     $ap->add_arg('--param', type => 'Pair', split => ',');
     $ns = $ap->parse_args(split(' ', '--param a=1,b=2,c=3'));

     say $ns->param->{a}; # 1
     say $ns->param->{b}; # 2
     say $ns->param->{c}; # 3

     # You can use choice to restrict values
     $ap->add_arg('--env', choices => [ 'dev', 'prod' ],);

     # or use case-insensitive choices
     # Override the previous option with reset
     $ap->add_arg('--env', choices_i => [ 'dev', 'prod' ], reset => 1);

     # or use a coderef
     # Override the previous option
     $ap->add_args(
            '--env',
            choices => sub {
                    die "--env invalid values" if $_[0] !~ /^(dev|prod)$/i;
            },
        reset => 1,
     );

     # subcommands
     $ap->add_subparsers(title => 'subcommands'); # Must be called to initialize subcommand parsing
     $list_parser = $ap->add_parser(
             'list',
             help => 'List directory entries',
             description => 'A multiple paragraphs long description.',
     );

     $list_parser->add_args(
       [
         '--verbose', '-v',
          type => 'Count',
          help => 'Verbosity',
       ],
       [
         '--depth',
          help => 'depth',
       ],
     );

     $ns = $ap->parse_args(split(' ', 'list -v'));

     say $ns->current_command(); # current_command stores list,
                                 # Don't use this name for your own option

     $ns =$ap->parse_args(split(' ', 'help list')); # This will print the usage for the list command
     # help subcommand is automatically added for you
     say $ns->help_command(); # list

     # Copy parsing
     $common_args = Getopt::ArgParse->new_parser();
     $common_args->add_args(
       [
         '--dry-run',
          type => 'Bool',
          help => 'Dry run',
       ],
     );

     $sp = $ap->add_parser(
       'remove',
       aliases => [qw(rm)],           # prog remove or prog rm
       parents => [ $command_args ],  # prog rm --dry-run
     );

     # Or copy explicitly
     $sp = $ap->add_parser(
       'copy',
       aliases => [qw(cp)],           # prog remove or prog rm
     );

     $sp->copy_args($command_parser); # You can also copy_parsers() but in this case
                                      # $common_parser doesn't have subparsers

DESCRIPTION
    Getopt::ArgParse, Getopt::ArgParse::Parser and related classes together
    aim to provide user-friendly interfaces for writing command-line
    interfaces. A user should be able to use it without looking up the
    document most of the time. It allows applications to define argument
    specifications and it will parse them out of @AGRV by default or a
    command line if provided. It implements both named arguments, using
    Getopt::Long for parsing, and positional arguments. The class also
    generates help and usage messages.

    The parser has a namespace property, which is an object of
    ArgParser::Namespace. The parsed argument values are stored in this
    namespace property. Moreover, the values are stored accumulatively when
    parse_args() is called multiple times.

    Though inspired by Python's argparse and names and ideas are borrowed
    from it, there is a lot of difference from the Python one.

  Getopt::ArgParser::Parser
    This is the underlying parser that does the heavylifting.

    Getopt::ArgParse::Parser is a Moo class.

   Constructor
      my $parser = Getopt::ArgParse->new_parser(
        help        => 'short description',
        description => 'long description',
      );

    The former calls Getopt::ArgParser::Parser->new to create a parser
    object. The parser constructor accepts the following parameters.

    All parsers are created with a predefined Bool option --help|-h. The
    program can choose to reset it, though.

    *       prog

            The program's name. Default $0.

    *       help

            A short description of the program.

    *       description

            A long description of the program.

    *       namespace

            An object of Getopt::ArgParse::Namespace. An empty namespace is
            created if not provided. The parsed values are stored in it, and
            they can be referred to by their argument names as the
            namespace's properties, e.g. $parser->namespace->boo. See also
            Getopt::ArgParse::Namespace

    *       parser_configs

            The Getopt::Long configurations. See also Getopt::Long

    *       parents

            Parent parsers, whose argument and subparser specifications the
            new parser will copy. See copy() below

    *       error_prefix

            Customize the message prefixed to error messages thrown by
            Getopt::ArgParse, default to 'Getopt::ArgParse: '

    *       print_usage_if_help

            Set this to false to not display usage messages even if --help
            is on or the subcommand help is called. The default behavior is
            to display usage messages if help is set.

   add_arg, add_argument, add_args, and add_arguments
      $parser->add_args(
        [ '--foo', required => 1, type => 'Array', split => ',' ],
        [ 'boo', required => 1, nargs => '+' ],
      );

    The object method, arg_arg or the longer version add_argument, defines
    the specification of an argument. It accepts the following parameters.

    add_args or add_arguments() is to add multiple arguments.

    *       name or flags

            Either a name or a list of option strings, e.g. foo or -f,
            --foo.

            If dest is not specified, the name or the first option without
            leading dashes will be used as the name for retrieving values.
            If a name is given, this argument is a positional argument.
            Otherwise, it's a named argument.

            Hyphens can be used in names and flags, but they will be
            replaced with underscores '_' when used as option names. For
            example:

                $parser->add_argument( [ '--dry-run', type => 'Bool' ]);
                # command line: prog --dry-run
                $parser->namespace->dry_run; # The option's name is dry_run

            A name or option strings are following by named parameters.

    *       dest

            The name of the attribute to be added to the namespace populated
            by parse_args().

    *       type => $type

            Specify the type of the argument. It can be one of the following
            values:

            *       Scalar

                    The option takes a scalar value.

            *       Array

                    The option takes a list of values. The option can appear
                    multiple times in the command line. Each value is
                    appended to the list. It's stored in an arrayref in the
                    namespace.

            *       Pair

                    The option takes a list of key-value pairs separated by
                    the equal sign '='. It's stored in a hashref in the
                    namespace.

            *       Bool

                    The option does not take an argument. It's set to true
                    if the option is present or false otherwise. A 'no_bool'
                    option is also available, which is the negation of
                    bool().

                    For example:

                        $parser->add_argument('--dry-run', type => 'Bool');

                        $ns = $parser->parse_args(split(' ', '--dry-run'));

                        print $ns->dry_run; # true
                        print $ns->no_dry_run; # false

            *       Count

                    The option does not take an argument and its value will
                    be incremented by 1 every time it appears on the command
                    line.

    *       split

            split should work with types 'Array' and 'Pair' only.

            split specifies a string by which to split the argument string
            e.g. if split => ',', a,b,c will be split into [ 'a', 'b', 'c'
            ].When split works with type 'Pair', the parser will split the
            argument string and then parse each of them as pairs.

    *       choices or choices_i

            choices specifies a list of the allowable values for the
            argument or a subroutine that validates input values.

            choices_i specifies a list of the allowable values for the
            argument, but case insensitive, and it doesn't allow to use a
            subroutine for validation.

            Either choices or choices_i can be present or completely
            omitted, but not both at the same time.

    *       default

            The value produced if the argument is absent from the command
            line.

            Only one value is allowed for scalar argument types: Scalar,
            Count, and Bool.

    *       required

            Whether or not the command-line option may be omitted (optionals
            only). This has no effect on types 'Bool' and 'Count'. An
            optional option is marked by the question mark ? in the
            generated usage, e.g. --help, -h ? show this help message and
            exit

            This parameter is ignored for Bool and Count types for they will
            already have default values.

    *       help

            A brief description of what the argument does.

    *       metavar

            A name for the argument in usage messages.

    *       reset

            Set reset to override the existing definition of an option. This
            will clear the value in the namspace as well.

    *       nargs - Positional option only

            This only instructs how many arguments the parser consumes. The
            program still needs to specify the right type to achieve the
            desired result.

            *       n

                    1 if not specified

            *       ?

                    1 or 0

            *       +

                    1 or more

            *       *

                    0 or many. This will consume the rest of arguments.

   parse_args
      $namespace = $parser->parse_args(@command_line);

    This object method accepts a list of arguments or @ARGV if unspecified,
    parses them for values, and stores the values in the namespace object.

    A few things may be worth noting about parse_args().

    First, parsing for named Arguments is done by Getopt::Long

    Second, parsing for positional arguments takes place after that for
    named arguments. It will consume what's still left in the command
    line.

    Finally, the Namespace object is accumulatively populated. If
    parse_args() is called multiple times to parse a number of command
    lines, the same namespace object is accumulatively populated. For Scalar
    and Bool options, this means the previous value will be overwritten.
    For Pair and Array options, values will be appended. And for a Count
    option, it will add on top of the previous value.

    In face, the program can choose to pass an already populated namespace
    when creating a parser object. This is to allow the program to pre-load
    values to a namespace from conf files before parsing the command line.

    And finally, it does NOT display usage messages if the argument list is
    empty. This may be contrary to many other implementations of argument
    parsing.

   argv
      @argv = $parser->argv; # called after parse_args

    Call this after parse_args() is invoked to get the unconsumed arguments.
    It's up to the application to decide what to do if there is a surplus of
    arguments.

   The Namespace Object
    The parsed values are stored in a namespace object. Any class with the
    following three methods:

      * A constructor new()
      * set_attr(name => value)
      * get_attr(name)

    can be used as the Namespace class.

    The default one is Getopt::ArgParse::Namespace. It uses autoload to
    provide a readonly accessor method using dest names to access parsed
    values. However, this is not required for user-defined namespace. So
    within the implementation, $namespace->get_attr($dest) should always be
    used.

  Subcommand Support
    Note only one level of subcommand parsing is supported. Subcommands
    cannot have subcommands.

    Call add_subparsers() first to initialize the current parser for
    subcommand support. A help subcommand is created as part of the
    initialization. The help subcommand has the following options:

        required positional arguments:
             COMMAND      ? Show the usage for this command
        optional named arguments:
            --help, -h     ? show this help message and exit
            --all, -a      ? Show the full usage

    Call add_parser() to add a subparser for each subcommand. Use the parser
    object returned by add_parser() to add the options to the subcommand.

    Once subcommand support is on, if the first argument is not a flag, i.e.
    starting with a dash '-', the parser's parse_args() will treat it as a
    subcommand. Otherwise, the parser parses for the defined arguments.

    The namespace's current_command() will contain the subcommand after
    parsing successfully.

    Unlike arguments, subparsers cannot be reset.

   add_subparsers
      $parser->add_subparsers(
        title       => 'Subcommands',
        description => 'description about providing subcommands',
      );

    add_subparsers must be called to initialize subcommand support.

    *       title

            A title message to mark the beginning of subcommand usage in the
            usage message

    *       description

            A general description appearing about the title

   add_parser
      $subparser = $parser->add_parser(
         'list',
         aliases     => [qw(ls)],
         help        => 'short description',
         description => 'a long one',
         parents => [ $common_args ], # inherit common args from
                                      # $common_args
      );

    *       $command

            The first argument is the name of the new command.

    *       help

            A short description of the subcommand.

    *       description

            A long description of the subcommand.

    *       aliases

            An array reference containing a list of command aliases.

    *       parents

            An array reference containing a list of parsers whose
            specification will be copied by the new parser.

  get_parser
       $subparser = $parser->get_parser('ls');

    Return the parser for parsing the $alias command if exists.

  Copying Parsers
    A parser can copy argument specification or subcommand specification for
    existing parsers. A use case for this is that the program wants all
    subcommands to have a command set of arguments.

   copy_args
       $parser->copy_args($common_args_parser);

    Copy argument specification from the $parent parser

   copy_parsers
       $parser->copy_parsers($common_args_parser);

    Copy parser specification for subcommands from the $parent parser

   copy
       $parser->copy($common_args_parser);

    Copy both arguments and subparsers.

  Usage Messages and Related Methods
   format_usage
      $usage = $parser->format_usage;

    Return the formatted usage message for the whole program in an array
    reference.

   print_usage
       $parser->print_usage;

    Print the usage message returned by format_usage().

   format_command_usage
      $usage = $parser->format_command_usage($subcommand);

    Return the formatted usage message for the command in an array reference.

   print_command_usage
      $parser->print_command_usage($subcommand);

    Print the usage message returned by format_command_usage(). If $command
    is not given, it will first try to use $self->namespace->help_command,
    which will be present for the help subcommand, and then
    $self->namespace->current_command.

   
SEE ALSO
    Getopt::Long
    Python's argparse

AUTHOR
    Mytram <mytram2@gmail.com> (original author)

COPYRIGHT AND LICENSE
    This software is Copyright (c) 2014 by Mytram.

    This is free software, licensed under:

      The Artistic License 2.0 (GPL Compatible)
