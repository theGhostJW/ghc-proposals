Rich error messages from GHC
============================

.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.
.. ticket-url:: Leave blank. This will eventually be filled with the
                ticket URL which will track the progress of the
                implementation of the feature.
.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. sectnum::
.. contents::

Extend GHC's internal error message representation and expose it to tooling consumers.


Motivation
------------
Currently IDE tools (e.g. `Haskell IDE Engine
<https://github.com/haskell/haskell-ide-engine>`_) have to rely parsing GHC's
human-readable error messages to glean even a basic understanding of the nature
of the error. Not only is this parsing fragile, it also severely limits the
sorts of information the tool can gather about the error. This is unfortunate
since, as GHC ventures further into the territory of dependent types, error
messages grow in complexity and size.

Other languages (e.g. Idris, with their Emacs mode) have `demonstrated
<https://www.youtube.com/watch?v=m7BBCcIDXSg>`_ that enriching error message
representations with embedded AST elements can open the door to useful
interactions between the user, IDE tools, and the compiler. Let's open this
door.

Note that this proposal is **not** about changing the default presentation of
error messages produced by the ``ghc`` executable. Rather, it merely discusses
the plumbing necessary to enable downstream consumers (e.g. REPL shells, editor
extensions, and language servers) to make these changes on their own.

Also see `GHC #8809 <https://gitlab.haskell.org/ghc/ghc/issues/8809>`_.


Proposed Change Specification
-----------------------------
Error messages in GHC are currently represented as simple pretty-printer
documents ::

    -- | A pretty-printer document
    data SDoc

    -- | An error message
    type ErrMsg = SDoc

We propose to refactor this into ::

    -- | A pretty-printer document
    data SDoc a

    -- | An error message
    type ErrMsg = SDoc ErrorMessageItem

In this scheme ``SDoc`` would be a monadic-style pretty-printer document as
provided by ``wl-ppprint-extras``.

The ``ErrorMessageItem`` type would be a sum type including a variety of
elements frequently found in error messages that tooling users would find
useful to have available in structured form. There are a number of things that
might be included in this type but the initial cases we propose here fall into
a few categories which we will address below.

Error message metadata
~~~~~~~~~~~~~~~~~~~~~~

This captures some metadata typical of errors ::

    data ErrorMessageItem
      = EWarningHeader
            { srcSpan    :: SrcSpan
            , warnFlag   :: Maybe WarningFlag
              -- ^ Which warning flag can be used 
              -- to disable the warning
            }

Haskell AST elements
~~~~~~~~~~~~~~~~~~~~

These are the elements of the program we are compiling. For instance ::

    data ErrorMessageItem
      = ...
      | ESrcSpan SrcSpan  -- A source span
      | EIdentifier Id    -- An identifier
      | EType       Type  -- An identifier

Error message idioms
~~~~~~~~~~~~~~~~~~~~

In addition, we can also capture common idioms found in error messages. For
instance, consider the case of the all-too-frequent expected-actual error ::

.. code-block:: none

    Test.hs:7:7: error:
        • Couldn't match expected type ‘Int’ with actual type ‘[Char]’
        • In the first argument of ‘f’, namely ‘"hi"’
          In the expression: f "hi"
          In an equation for ‘g’: g = f "hi"
   
This could be represented as ::

    data ErrorMessageItem
      = ...
      | EExpectedActual { expectedType :: Type -- ^ what the typechecker expected
                        , actualType   :: Type -- ^ what the typechecker actually found
                        }

Likewise, the message,

.. code-block:: none

    hi.hs:5:5: error:
        • Variable not in scope: foldl'
        • Perhaps you meant one of these:
            ‘foldl’ (imported from Data.Foldable),
            ‘foldl1’ (imported from Prelude), ‘foldr’ (imported from Prelude)
          Perhaps you want to add ‘foldl'’ to the import list
          in the import of ‘Data.Foldable’ (hi.hs:3:1-28).

This could be represented as ::

    data ErrorMessageItem
      = ...
      | ENotInScope { badName               :: OccName
                    , suggestedAlternatives :: [Name]
                    }

Refactoring Actions
~~~~~~~~~~~~~~~~~~~

Additionally, we could further include more action-oriented items. For
instance, in numerous places GHC suggests enabling a language extension:

.. code-block:: none

    hi.hs:8:33: error:
        Illegal operator ‘+’ in type ‘n + 1’
          Use TypeOperators to allow operators in types

This could be represented as ::

    data ErrorMessageItem
      = ...
      | ESuggestExtension LanguageExtension

Likewise, suggestions of changes to ``import`` statements, e.g.

.. code-block:: none

    hi.hs:5:5: error:
        • Variable not in scope: foldl'
        • Perhaps you meant one of these:
            ‘foldl’ (imported from Data.Foldable),
            ‘foldl1’ (imported from Prelude), ‘foldr’ (imported from Prelude)
          Perhaps you want to add ‘foldl'’ to the import list
          in the import of ‘Data.Foldable’ (hi.hs:3:1-28).

can be encoded as ::

    data ErrorMessageItem
      = ...
      | ESuggestAddedImport SrcSpan Name  -- source span of import statement
                                          -- and suggested Name to import


An Example
~~~~~~~~~~

In general error messages will be built from plain pretty-printer documents
with embedded ``ErrorMessageItem``\s. For instance, consider the error

.. code-block:: none

    hi.hs:5:5: error:
        • Variable not in scope: foldl'
        • Perhaps you meant one of these:
            ‘foldl’ (imported from Data.Foldable),
            ‘foldl1’ (imported from Prelude), ‘foldr’ (imported from Prelude)
          Perhaps you want to add ‘foldl'’ to the import list
          in the import of ‘Data.Foldable’ (hi.hs:3:1-28).

This might be built by GHC as ::

    pure (EErrorHeader $span Nothing)
    <> pure (ENotInScope $foldl') [ $foldl, $foldl1 ]
    <> pure (ESuggestAddedImport $import_span $foldl') [ $foldl, $foldl1 ]

where ``$foo`` denotes the GHC AST item for ``foo`` and ``pure`` lifts an
``ErrorMessageItem`` into an ``SDoc``::

    pure :: ErrorMessageItem -> SDoc ErrorMessageItem

Effect and Interactions
-----------------------
By introducing rich semantic content into error messages and exposing these
documents via the GHC API, we allow tooling authors significantly more
flexibility in presenting (and automatically fixing) compile-time errors.
We list a few compelling applications below (roughly in order of complexity):

 * A REPL front-end might implement color-coded output, choosing a token's
   color by its syntactic class (e.g. type constructor, data constructor, or
   identifier), its name (e.g. all occurrences of ``foldl`` shown in red,
   occurrences of ``concat`` shown in blue), or some other criterion entirely.

 * A REPL front-end or IDE tool might allow users the ability to interactively
   navigate a type in a type error and, for instance, allow the user to
   interactively expand type synonyms, show kind signatures, etc.

 * An IDE tool might ask GHC to defer expensive analyses typically done
   during error message construction (e.g. `computing valid hole fits
   <https://gitlab.haskell.org/ghc/ghc/issues/16875#note_210045>`_) and instead
   query GHC for the analysis result asynchronously (or even only when
   requested by the user), shrinking the edit/typechecking iteration time.

 * An IDE tool might use the action-items (e.g. ``ESuggestExtension`` and
   ``ESuggestAddedImport`` above) to present automated refactoring options to
   the user.


Costs and Drawbacks
-------------------

Judging from a prototype implementation undertaken a few years ago, the impact
of embedding structured data instead of producing pretty-printer documents is
quite minimal. The idioms which we are trying to represent are implemented
in helper functions in ``TcErrors``, anyways.

One unexpected challenge in implementing the prototype was the difficulty of 
finding or adapting a pretty-printer library with the desired monadic
annotation semantics that does not break the formatting of GHC's error message
output. A previous attempt at using the ``prettyprinter`` library `found
<https://github.com/quchen/prettyprinter/issues/34>` that GHC's error messages
generally include a great deal of superfluous whitespace which is eliminated by
the ``pretty`` library yet not by most other libraries.

The greatest challenge in this proposal is designing a vocabulary of
``ErrorMessageItem``\s that can be usefully and unambiguously interpreted by
error message consumers. We propose a few simple items in the design discussion
above, but we only scratch the surface of what could be encoded and what might
be useful. We hope that the discussion that arises from this proposal will shed
light on additional items. Moreover, we anticipate that the vocabulary will
grow in time as new tooling applications are found.


Alternatives
------------
There are a few alternatives:

 * Continuing representing error messages as plain pretty-printer documents.
   We think this would be a shame as it would 

 * Represent error messages as fully structured data using a large sum
   type. Core GHC contributors have in the past opposed this approach on
   account of maintanence difficulty. We agree and further think that the
   proposal laid out above can capture most of the precision of a fully
   structured representation with a fraction of the maintanence overhead.


Unresolved Questions
--------------------

As described in the "Costs and Drawbacks" section above, a number of questions
regarding the design of the ``ErrorMessageItem`` type remain open.



Implementation Plan
-------------------

Well-Typed LLP will implement this proposal with financial support from
Richard Eisenberg.

