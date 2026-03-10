# UI_Better
An improved version of the Perl module Term::UI

## See also

 - [The problem with Term::UI](https://gr33nonline.wordpress.com/2026/03/10/the-problem-with-termui/)

## Notes

### Term::UI bugs

There is a bug or two in the Term::UI:

 - History inconsistency – index or selection
 - Duplicate history - two entries for each input

#### Bug#1: History is incorrect/inconsistant

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

Fix: Move the following three lines

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

#### Bug#2: Duplicate entries

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

If you combine these two commented out lines with the fix above for **Bug#1**, then the behaviour of the history is as expected and desired – tht is to say that the history only shows the actual selection and never the index.


### Effecting the changes

Make duplicate copies of the `UI.pm` and `UI/History.pm` files and rename them `UI_Better.pm` and `UI_Better/History_Better.pm` respectively. Remember to rename the package references within the files as well.

However, duplicating `readline.pm` and creating `ReadLine_Better/readline_Better.pm` is a little more complex.

Surely, there should be a standard way of invoking `ReadLine` and disabling the history – without the need to "hack" the module itself?

### Disabling the "autohistory"


See [Term::ReadLine - I need to hit the up arrow twice to retrieve history](https://stackoverflow.com/q/13332908/4424636) which pretty much describes **Bug#2** above, the duplicate call to `$term->addhistory()`.

However, this "duplicate" call, in `UI.pm` would be fine, if `$term->Features->{autohistory}` is turned off.

The answer to the above question mentions `$term->Features->{autohistory}` being true. But ow does one set it to false?

Note that this does not work, from [Disabling autohistory in Term::ReadLine
](https://github.polettix.it/ETOOBUSY/2020/09/24/term-readline-autohistory/)


```none
# This disables autohistory
$term->MinLine();
```

However, from [Term::ReadLine::Gnu vs. history control](https://www.perlmonks.org/?node_id=1007439), setting to a high value effectivly disables the `readline` history, thereby allowing the UI history to operte as expected.

```none
# This disables autohistory
$term->MinLine(MAX);
```

So, the only changes to the modules themselves required are to `UI.pm`.

A fixed verion of `UI.pm` called `UI_Better.pm` is included in this repository. 

For the ease of changing the behaviour of `UI_Better`, there are two flag near the top of the file:

```none
our $flg_post_add_history = 1;  # Move the add history to the end of _tt_readline()
our $flg_no_history = 0;        # Disable all history calls in _tt_readline()
```
