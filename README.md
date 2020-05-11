# ABNF

![abnf-tox](https://github.com/declaresub/abnf/workflows/abnf-tox/badge.svg)


ABNF is a package that generates parsers from ABNF grammars.  The main purpose of this
package is to parse data as specified in RFCs.  But it should be able to handle any ABNF 
grammar.

ABNF was originally written a few years ago for parsing HTTP headers in a web framework.
The code herein has been in use in production here and there on the internet since then.


## Requirements

ABNF has been tested with Python 3.5-8. The package has no other dependencies.

## Installation

The abnf package is available from [PyPI](https://pypi.org/project/abnf/).

Install it in the usual way.

    pip install abnf

### Verification

The abnf package is signed with GPG.  The public key is available from [github](https://github.com/declaresub.gpg),
or [OpenPGP](https://keys.openpgp.org/).  The key fingerprint is `3A27 290F D243 BD83 BC3F  5BC8 86C0 57F9 6A41 A77B`.

Once you have imported the public key into GPG, you can check the signature by downloading
the files and the signature files from [PyPI](https://pypi.org/project/abnf/).  No
download links for the signature files are present; you need to create them by appending `.asc`
to the package URLs.  

Once downloaded, use gpg to verify the signatures.

    gpg --verify abnf-1.0.0.tar.gz.asc abnf-1.0.0.tar.gz
    gpg --verify abnf-1.0.0-py2.py3-none-any.whl.asc abnf-1.0.0-py2.py3-none-any.whl


## Usage

The main class of abnf is Rule.  You should think of a Rule subclass as corresponding to an
ABNF grammar.  Then instances of that subclass represent the rules of that grammar.

The Rule class is initialized with the core ABNF rules OCTET, BIT, HEXDIG, CTL, HTAB, LWSP, 
CR, VCHAR, DIGIT, WSP, DQUOTE, LF, SP, CRLF, CHAR, ALPHA, and so are available in any subclass
of Rule.

Create a Rule object using the class method create.

    rule = Rule.create('double-quoted-string = DQUOTE *(%x20-21 / %x23-7E / %x22.22) DQUOTE')

To later retrieve the object just created:

    rule = Rule('double-quoted-string')

Rule objects are cached, so `Rule('double-quoted-string')` should always return the same object,
though you might not want to depend on that.

ABNF includes several grammars.  The Rule subclass ABNFGrammarRule implements the rules 
for ABNF.  The package `abnf.grammars` includes grammars from several RFCs.

    from abnf.grammars import rfc7232
    src = 'W/"moof"'
    node, start = rfc7232.Rule('ETag').parse(src)
    print(str(node))
    
The output is

    Node(
        name=ETag, 
        children=
            [
            Node(
                name=entity-tag, 
                children=
                    [
                    Node(
                        name=weak, 
                        children=
                            [
                            Node(
                                name=literal, 
                                offset=0, 
                                value="W/"
                                )
                            ]
                        ), 
                        Node(
                            name=opaque-tag, 
                            children=
                                [
                                Node(
                                    name=DQUOTE, 
                                    children=
                                        [
                                        Node(
                                            name=literal, 
                                            offset=2, 
                                            value="""
                                            )
                                        ]
                                    ), 
                                Node(
                                    name=etagc, 
                                    children=
                                        [
                                        Node(
                                            name=literal, 
                                            offset=3, 
                                            value="m"
                                            )
                                        ]
                                    ), 
                                Node(
                                    name=etagc, 
                                    children=
                                        [
                                        Node(
                                            name=literal, 
                                            offset=4, 
                                            value="o"
                                            )
                                        ]
                                    ), 
                                Node(
                                    name=etagc, 
                                    children=
                                        [
                                        Node(
                                            name=literal, 
                                            offset=5, 
                                            value="o"
                                            )
                                        ]
                                    ), 
                                Node(
                                    name=etagc, 
                                    children=
                                    [
                                    Node(
                                        name=literal, 
                                        offset=6, 
                                        value="f"
                                        )
                                    ]
                                ), 
                            Node(
                                name=DQUOTE, 
                                children=
                                    [
                                    Node(
                                        name=literal, 
                                        offset=7, 
                                        value="""
                                        )
                                    ]
                                )
                            ]
                        )
                    ]
                )
            ]
        )'
    
    


The modules in `abnf.grammars` may serve as an example for writing other Rule subclasses. 
In particular, some of the RFC grammars incorporate rules by reference from other RFC. 
`abnf.grammars.rfc7230` shows a way to import rules from another Rule subclass.

ABNF uses CRLF as a delimiter for rules.  Because many text editors (e.g. BBEdit) substitute line endings 
without telling the user, ABNF expects preprocessing of grammars into python lists of rules as 
in `abnf.grammars`.

### Errors

abnf implements two exception subclasses, ParseError and GrammarError.  

A GrammarError is raised when parsing encounters an undefined rule.  

A ParseError is raised when parsing fails for some reason.  Error reporting is nothing
more than a stack trace, but that usually allows one to get to the source of the problem.


## Examples

### Validate an email address

The code below validates an arbitrary email address.  If src is not syntactically valid,
a ParseError is raised.

    from abnf.grammars import rfc5322
    
    src = 'test@example.com'
    parser = rfc5322.Rule('address')
    parser.parse_all(src)

### Extract the actual address from an email address

    from abnf.grammars import rfc5322

    def get_address(node):
        """Do a breadth-first search of the tree for addr-spec node.  If found, 
        return its value."""
        queue = [node]
        while queue:
            n, queue = queue[0], queue[1:]
            if n.name == 'addr-spec':
                return n.value
            else:
                queue.extend(n.children)
        return None

    src = 'John Doe <jdoe@example.com>'
    parser = rfc5322.Rule('address')
    node = parser.parse_all(src)
    address = get_address(node)
    
        
    for x in node_iterator(node):
        if x.name == 'addr-spec':
            print(x.value)
            break


### Extract authentication information from an HTTP Authorization header.

    from abnf.parser import NodeVisitor
    from abnf.grammars import rfc7235

    header_value = 'Basic YWxhZGRpbjpvcGVuc2VzYW1l'
    parser = rfc7235.Rule('Authorization')
    node, offset = parser.parse(header_value, 0)

    class AuthVisitor(NodeVisitor):
        def __init__(self):
            super().__init__()
            self.auth_scheme = None
            self.token = None

        def visit_authorization(self, node):
            for child_node in node.children:
                self.visit(child_node)

        def visit_credentials(self, node):
            for child_node in node.children:
                self.visit(child_node)

        def visit_auth_scheme(self, node):
            self.auth_scheme = node.value

        def visit_token68(self, node):
            self.token = node.value

    visitor = AuthVisitor()
    visitor.visit(node)
    
The result is that visitor.auth_scheme = 'Basic', and visitor.token = 'YWxhZGRpbjpvcGVuc2VzYW1l'

## Implementation

abnf is implemented using parser combinators. There is a class Literal whose instances are
initialized with either a string like 'moof', or a tuple like ('a', 'z') representing a range.
The result is a parser that can match the initialized value.

ABNF operations -- alternation, concatenation, repeat, etc. are implemented as classes.  
For example, Alternation(Literal('foo'), Literal('bar')) returns a parser that implements the
ABNF expression 

    "foo" / "bar"
 
Alternation is implemented to do longest match. In the event of a tie, the first
match is returned.
   
The whole mess is bootstrapped by writing out the parsers for the grammar and core rules 
by hand.  The ABNFGrammarRule class represents the ABNF grammar, and is used to parse other
grammars.  It is also capable of parsing its own grammar.

        
## Development, Testing, etc.

Should you wish to tinker with the code, install in a virtual environment and have at it.
The file requirements.txt contains the packages I use for testing and such.

A good starting point would be to run pytest and see that all tests pass.

    pytest --cov-report term-missing --cov=abnf 

The test suite includes fuzz testing with test data generated using [abnfgen](http://www.quut.com/abnfgen/).
Some of the test rules are long and gruesome.  Thus the tests take a bit of time to complete.
Skip the fuzz tests with 

        pytest --cov-report term-missing --cov=abnf --ignore=tests/fuzz

Following changes, run 

    pylint abnf

to resolve any problems found, then 

    tox
    
to execute tests for python 3.5-3.8.

The code is formatted using black.

    black src/abnf

