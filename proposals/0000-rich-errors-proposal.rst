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
sorts of information the tool can gather about the error.

Other languages (e.g. Idris, with their Emacs mode) have `demonstrated
<https://www.youtube.com/watch?v=m7BBCcIDXSg>`_ that enriching error message
representations with embedded AST elements can open the door to useful
interactions between the user, IDE tools, and the compiler. Let's open this
door.

Also see `GHC #8809 <https://gitlab.haskell.org/ghc/ghc/issues/8809>`_.


Proposed Change Specification
-----------------------------
Error messages in GHC are currently represented as simple pretty-printer documents ::

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

The ``ErrorMessageItem`` type would be a sum type including a number of AST
elements frequently found in error messages. Initially, we propose the
following ::

    data ErrorMessageItem
      = ESrcSpan SrcSpan  -- A source span
      | EIdentifier Id    -- An identifier
      | EType       Type  -- An identifier

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
      | EExpectedActual Type Type
                          -- A pair of a type that the typechecker expected to find
                          -- and a 

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

Likewise, suggestions of changes to ``import`` statements:

.. code-block:: none

    hi.hs:5:5: error:
        • Variable not in scope: foldl'
        • Perhaps you meant one of these:
            ‘foldl’ (imported from Data.Foldable),
            ‘foldl1’ (imported from Prelude), ‘foldr’ (imported from Prelude)
          Perhaps you want to add ‘foldl'’ to the import list
          in the import of ‘Data.Foldable’ (hi.hs:3:1-28).

Can be encoded as ::

    data ErrorMessageItem
      = ...
      | ESuggestAddedImport SrcSpan Name  -- source span of import statement
                                          -- and suggested Name to import


Effect and Interactions
-----------------------
By introducing rich semantic content into error messages and exposing these
documents via the GHC API, we allow tooling authors significantly more
flexibility in presenting (and automatically fixing) compile-time errors:

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
Give an estimate on development and maintenance costs. List how this effects learnability of the language for novice users. Define and list any remaining drawbacks that cannot be resolved.


Alternatives
------------
List existing alternatives to your proposed change as they currently exist and discuss why they are insufficient.


Unresolved Questions
--------------------
Explicitly list any remaining issues that remain in the conceptual design and specification. Be upfront and trust that the community will help. Please do not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to the steering committee.


Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other ressources and prerequisites are required for implementation?

