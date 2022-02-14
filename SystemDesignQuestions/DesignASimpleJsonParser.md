# Writing a simple JSON parser

## What parsing is and (typically) is not

Parsing is often broken up into two stages: 

     1. lexical analysis and 
     2. syntactic analysis.
     

>*Lexical analysis breaks source input into the simplest decomposable elements of a language called "tokens". Syntactic analysis (often itself called "parsing") receives the list of tokens and tries to find patterns in them to meet the language being parsed.*

>Parsing does not determine semantic viability of an input source. Semantic viability of an input source might include whether or not a variable is defined before being used, whether a function is called with the correct arguments, or whether a variable can be declared a second time in some scope.

### Lexical analysis

Lexical analysis breaks down an input string into tokens. Comments and whitespace are often discarded during lexical analysis so you are left with a simpler input you can search for grammatical matches during the syntactic analysis.

Assuming a simple lexical analyzer, you might iterate over all the characters in an input string (or stream) and break them apart into fundemental, non-recursively defined language constructs such as integers, strings, and boolean literals. In particular, strings must be part of the lexical analysis because you cannot throw away whitespace without knowing that it is not part of a string.




