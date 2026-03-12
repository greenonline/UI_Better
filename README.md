# UI_Better
An improved version of the Perl module `Term::UI`

## See also

 - [The problem with Term::UI](https://gr33nonline.wordpress.com/2026/03/10/the-problem-with-termui/)

## Notes

### Term::UI bugs

There is a bug or two in v0.50 of [`Term::UI`](https://perldoc.perl.org/5.12.2/Term::UI):

#### Bugs

Issues, and errors, arising from inconsistencies in functionality, duplicated output and unusual conditions:

 - History inconsistency – index or selection
 - Duplicate history - two entries for each input
 - If no default is specified, hitting Enter causes error due to uninitialised variable, `$answer` in `split()`

#### Enhancements

Some of the features of `Term::UI` are lacking the ability to fine-tune their functionality:

 - No option to check only the first word, of a multi-word input, against the valid choices provided
   - Every word, on a multi-word input, is checked for validity
 - No option to print choices on one line 
   - The choices are printed one-line per item, which can quickly "flood" the screen, if there are a large number of options
 - No option to suppress printing of the choices
   - The choices, if present, have to be display. The user may wish to suppress the automatic printing of these options, as they may be displayed elsewhere.

In other words:

 - Enhancement: Check only the first word for validity and return all words if the first is valid.
 - Enhancement: Add option to print choices on one line
 - Enhancement: Add option to suppress the automatic printing of the choices

### Bug#1: History is incorrect/inconsistant

The history shows the shortcut menu selections rather than the actual option selected.

For example, if you use the code from the synopsis: If you select the default option then the actual selection is shown in the history, i.e. "blue", when you hit the up arrow at the next prompt: 

```none

1> blue
2> red
3> green

What is your favourite colour? [1]:                                             
Do you like cookies? [Y/n]: blue               
```

However, if you chose anything other than the default then the option number is saved to the history instead, i.e. "3":

```none

1> blue
2> red
3> green

What is your favourite colour? [1]: 3                                           
Do you like cookies? [Y/n]: 3                                                   
```

The history is saved here, in UI.pm, in `_tt_readline()`:

```none
        ### pose the question
        my $answer = defined $preput
          ? $term->readline($prompt, $preput)
          : $term->readline($prompt);

        $answer     = $default unless length $answer;

        $term->addhistory( $answer ) if length $answer;

        ### add both prompt and answer to the history
        history( defined $answer ? "$prompt $answer" : "$prompt", 0 );

```

#### Fix

Move the following three lines

```none
        $term->addhistory( $answer ) if length $answer;
        
        ### add both prompt and answer to the history
        history( defined $answer ? "$prompt $answer" : "$prompt", 0 );

```

 to the `else{}` block containing the return from `_tt_readline()` at the end of the function, like so:


```none
        } else {
            #$term->addhistory( $multi ? @rv : $rv[0] ) ;

            ### add both prompt and answer to the history
            #history( $multi ? @rv : $rv[0], 0 );

            return $multi ? @rv : $rv[0];
        }
    }
}
```

That way, any numeric options entered, have been converted to their actual values at that point of `return()`. Otherwise, prior to that point, only the default option is correctly added, and all other selections, made using their index, have their numeric index added to the history instead.

### Bug#2: Duplicate entries

On top of that, the module originally saves the history *twice*, if you do not select the default option. In other words, every history item is repeated twice, when the non-default option is selected. 

Using the simple example from the synopsis, the option 3 is entered twice. With the option number now replaced by the actual option, using the fix above, the issue is even clearer:

After one key press up

```none
1> blue
2> red
3> green

What is your favourite colour? [1]: 3                                           
Do you like cookies? [Y/n]: green                                               
```

After a second key press up

```none
1> blue
2> red
3> green

What is your favourite colour? [1]: 3                                           
Do you like cookies? [Y/n]: 3                                               
```

#### Further experimentation – Not really required

If you use this extended example from the synopsis (but with an aditional third question/response), with the original `UI.pm` code, the issue becomes clearer:

```none

use File::Basename;
use File::Spec;
use lib File::Spec->catdir(
            File::Basename::dirname(File::Spec->rel2abs($0)),
            '..',
            'lib');



use Term::UI;
#use Term::UI_Better;
use Term::ReadLine;

my $term = Term::ReadLine->new('brand');

my $reply = $term->get_reply(
                prompt => 'What is your favourite colour?',
                choices => [qw|blue red green|],
                default => blue,
);

my $bool = $term->ask_yn(
                    prompt => 'Do you like cookies?',
                    default => 'y',
            );

my $bool = $term->ask_yn(
                    prompt => 'Do you like salad?',
                    default => 'y',
            );


my $string = q[some_command -option --no-foo --quux='this thing'];

my ($options,$munged_input) = $term->parse_options($string);


### don't have Term::UI issue warnings -- default is '1'
$Term::UI::VERBOSE = 0;

### always pick the default (good for non-interactive terms)
### -- default is '0'
$Term::UI::AUTOREPLY = 1;

### Retrieve the entire session as a printable string:
$hist = Term::UI::History->history_as_string;
$hist = $term->history_as_string;
```

You require two up key presses to get to the preceding history item: When you get to the third question, press the up arrow repeatedly – there are two "n" then "green" and finally "3".

#### Where is the history set?

Even if you comment out each and every (there are 8) occurence of `history(` in `UI.pm`, the history is still saved, once. So it must be also be saved either by `Term::UI::History.pm`, or `Term::ReadLine`. 

FWIW, there are 32 occurences of `history` in UI.pm but these additional 24 occurences are in comments or POD.

However, there *appear* to be no "save to history" statements in UI::History.pm, only "history retrieval" functions. Looking in `Term::ReadLine::readline.pm`, again only apparently reteival functions exist. There is a third file `Term::ReadLine::Perl.pm`, but that has no occurences of `history(` and 17 occurences of `history`:

```none
#sub addhistory {}
*addhistory = \&AddHistory;

...

%features =  (appname => 1, minline => 1, autohistory => 1, getHistory => 1,
```

Is `autohistory` the issue? Changing it doesn't seem to have an effect, however.

#### The duplication is caused by `$term->addhistory()`

By accident, I noted that commenting out the line

```none
$term->addhistory( $answer ) if length $answer;
```

stopped the duplication of the history. 

Why are both of the following two lines in the code?

```none
        $term->addhistory( $answer ) if !$flg_no_history && !$flg_post_add_history && length $answer;

        ### add both prompt and answer to the history
        history( defined $answer ? "$prompt $answer" : "$prompt", 0 )  if !$flg_no_history && !$flg_post_add_history;
```

Shouldn't the history be added to only once? Or are there *two types* of "history"?

Using only the post add history block, and commenting out one of the two lines has differing effects. Entering "3" as the selection.

Commenting out only the first line, removes the duplicate entry, but leaves only the index in the history

```none
        } else {
            #$term->addhistory( $multi ? @rv : $rv[0] )  if !$flg_no_history && $flg_post_add_history;

            ### add both prompt and answer to the history
            history( $multi ? @rv : $rv[0], 0 )  if !$flg_no_history && $flg_post_add_history;

            return $multi ? @rv : $rv[0];
        }
```

Commenting out only the second line, removes the duplicate entries, but only because the second index in the history has been replaced by the actual selection by the line `$term->addhistory()`: 

```none
        } else {
            $term->addhistory( $multi ? @rv : $rv[0] )  if !$flg_no_history && $flg_post_add_history;

            ### add both prompt and answer to the history
            #history( $multi ? @rv : $rv[0], 0 )  if !$flg_no_history && $flg_post_add_history;

            return $multi ? @rv : $rv[0];
        }
```

In fact, the behaviour of commenting out only the second line is the same as not commenting out either, so the second line probably isn't the issue.

Commenting out both lines results in the index being saved only once, no duplicates. However, where is this index being saved?

#### In `readline.pm`

In `readline.pm` there is a function `add_line_to_history()`. 
It is called from two functions, in `F_AcceptLine()` and `F_SaveLine`. If you comment out the calls to `add_line_to_history()` in both `F_AcceptLine()` and `F_SaveLine` then no history is written and the issue of the written index goes away.

If you combine these two commented out lines with the fix above for **Bug#1**, then the behaviour of the history is as expected and desired – that is to say that the history only shows the actual selection and never the index.

### Bug#3 - Enter causes uninitialised variable error

If `default` is not specified, then an invalid input is entered, and then enter is pressed, and error occurs:

```none
Use of uninitialized value $answer in split at UI_Better.pm line 416.
```

This is fixed by adding, in `_tt_readline()`:

```none
        if (!defined $answer){
            $answer = "";
        }
```

after the line

```none
        $answer     = $default unless length $answer;
```

like so,

```none
        $answer     = $default unless length $answer;
        if (!defined $answer){
            $answer = "";
        }
```



### Bug#4 - Enhancement: Check only the first word for validity

Some code changes have been made to the conditional within the `LOOP` in `_tt_readline()` - an additional condition or two. 

If `first` is set true, then only the first word is checked, against the choices, for validity. If it is valid, then the first word and all of the other words are returned in an array. If the first word is invalid, then the user has to re-enter the input as before. This functionality also requires  `multi` to be set to `true`.

### Bug#5 - Enhancement: Print choices on one line

The code in `get_reply()` was modified to add a conditional for the `oneline` option, to `sprint` the choices, in a different one-line format, to `$args->{print_me}`.

### Bug#6 - Enhancement: Suppress printing of the choices

The code in `get_reply()` was modified to add yet another conditional for the `nochoices` option, to suppress all statements that `sprint` the choices to `$args->{print_me}`.

### Effecting the changes

Make duplicate copies of the `UI.pm` and `UI/History.pm` files and rename them `UI_Better.pm` and `UI_Better/History_Better.pm` respectively. Remember to rename the package references within the files as well.

However, duplicating `readline.pm` and creating `ReadLine_Better/readline_Better.pm` is a little more complex.

Surely, there should be a standard way of invoking `ReadLine` and disabling the history – without the need to "hack" the module itself?

### Disabling the "autohistory"


See [Term::ReadLine - I need to hit the up arrow twice to retrieve history](https://stackoverflow.com/q/13332908/4424636) which pretty much describes **Bug#2** above, the duplicate call to `$term->addhistory()`.

However, this "duplicate" call, in `UI.pm` would be fine, if `$term->Features->{autohistory}` is turned off.

The answer to the above question mentions `$term->Features->{autohistory}` being true. But how does one set it to false?

Note that the following does not work, from [Disabling autohistory in Term::ReadLine
](https://github.polettix.it/ETOOBUSY/2020/09/24/term-readline-autohistory/)

```none
# This disables autohistory
$term->MinLine();
```

However, from [Term::ReadLine::Gnu vs. history control](https://www.perlmonks.org/?node_id=1007439), setting to a high value effectivly disables the `readline` history, thereby allowing the UI history to operate as expected.

```none
# This disables autohistory
$term->MinLine(10000);
```

So, the only changes to the modules themselves required are to `UI.pm`.

### Provided files

A fixed verion of `UI.pm` called `UI_Better.pm` is included in this repository. 

For the ease of changing the behaviour of `UI_Better`, there are two flags near the top of the file:

```none
our $flg_post_add_history = 1;  # Move the add history to the end of _tt_readline()
our $flg_no_history = 0;        # Disable all history calls in _tt_readline()
```

The associated file, `UI_Better/History_Better.pm`, is actually identical (apart from the package name changes) to the original `History.pm` of [Term::UI](https://perldoc.perl.org/5.12.2/Term::UI) as is only included for completeness.

### New options

`get_reply()` now takes four additional optional Boolean arguments, `resolved`, `first`, `oneline` and `nochoices`. 

These optional arguments do the following:

 - add only item names (*not* item indices) to the history
 - only check the first word in a multiword answer, against the choices provided, respectively. These both default to `off`, or `false`.
 - display the choices on one line
 - suppress display of the choices altogether.


`$reply = $term->get_reply( prompt => 'question?', [choices => \@list, default => $list[0], multi => BOOL, resolved => BOOL, first => BOOL, oneline => BOOL, nochoices => BOOL, print_me => "extra text to print & record", allow => $ref] );`

 - `resolved` - add only fully resolved names to the history (instead of returning both items' indices and names)
 - `first` - check *only the first word* in a multi-word input against the valid choices (instead of all of the words), and return *all* words as an array.
 - `oneline` - display the choices as a comma separated list, on one line (instead of one line per choice). The indices are *not* displayed, although it is still possible to reference a choice via its index.
 - `nochoices` - Do not display the choices. The indices are also *not* displayed, although it is still possible to reference a choice via its index.

### Example usage

In the examples below, it should be noted that the default has been "disabled" by commenting out that line. If you require a default, then un-comment the line.

Multi-word input but check only first word against valid choices, add only choice item name to history (and not the choice index):

```none
  # Check only first word against valid choices
  # Multi-word input
  # Add only choice item name to history
  return $term->get_reply(
                  prompt => $myprompt,
                  choices => \@mychoices,
                  #default => $mydefault,
                  multi => 1,
                  resolved => 1,
                  first => 1,
                  print_me => $myprintme
              );
```

Display choices on one line, multi-word input (and check each word against valid choices), add only choice item name to history (and not the choice index):

```none
  # Display choices on one line
  # Multi-word input
  # Add only choice item name to history
  return $term->get_reply(
                  prompt => $myprompt,
                  choices => \@mychoices,
                  #default => $mydefault,
                  multi => 1,
                  resolved => 1,
                  oneline => 1,
                  print_me => $myprintme
              );
```

Display choices on one line, multi-word input but  check only first word against valid choices, add only choice item name to history (and not the choice index):

```none
  # Display choices on one line
  # Multi-word input
  # Check only first word against valid choices
  # Add only choice item name to history
  return $term->get_reply(
                  prompt => $myprompt,
                  choices => \@mychoices,
                  #default => $mydefault,
                  multi => 1,
                  resolved => 1,
                  first => 1,
                  oneline => 1,
                  print_me => $myprintme
              );
```

No choices displayed, multi-word input but check only the first word against valid choices, add only choice item name to history (and not the choice index):

```none
  # Suppress display of choices
  # Multi-word input
  # Check only first word against valid choices
  # Add only choice item name to history
  return $term->get_reply(
                  prompt => $myprompt,
                  choices => \@mychoices,
                  #default => $mydefault,
                  multi => 1,
                  resolved => 1,
                  first => 1,
                  nochoices => 1,
                  print_me => $myprintme
              );
```
