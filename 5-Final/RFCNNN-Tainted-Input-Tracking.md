---
RFC: NN
Author: Travis Plunk
Status: Draft
Version: 0.1
Area: Security
Comments Due: February 1, 2019
---

# Tainted Input Tracking

There is a need to add the ability to notify users when they use Tainted input (from an untrusted source)
that the input should be validated before it is used by sensitive cmdlets.

This was partly implemented in [#8257](https://github.com/PowerShell/PowerShell/pull/8257).
It added `[ValidateTrustedData]` which currently marks all input from constrained language mode as tainted and
throws an error if it is used by a property in full language mode.

Tainted input is tracked via a [ConditionalWeakTable](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.conditionalweaktable-2?view=netcore-2.1).
This raises so issues which are discussed in [Issues](#issues).

## Motivation

As a script module author, I can use PowerShell and be confident that the input I am using is trusted in constrained language mode so that I can avoid accidentally using untrusted input.

## Specification

### Preface

Each behavior was intended to be independent.
I also tried to prioritize the behaviors.
The intent is not that we would implement them all at once,
but implement what is reasonable,
perhaps behind a experimental flag and
then add more after getting feedback.
If there are any dependencies between behaviors,
the RFC should be updated with the dependency.

### Behaviors

1. Explicit Untaint

   * We should have an implementation that can be used in full language that allow the module author to explicitly untaint an object during parameter validation.
    This would allow validated input to be untainted and used.

     This can be one or both of the following implementations.

   1. Validation Attributes

      * Updated the validation attributes with a `bool` property,
         `MarkAsTrusted` that indicates that the validation also untaints the input.
         The engine would have to make sure that this is only done in Full Language mode.
         The engine would have to be updated to call validation attributes before performing the `[ValidateTrustedData]` operation.

   1. Method

      * A method to untaint an object.
         Because methods cannot be called in constrained language mode,
         the engine would not have to do anything to make sure this is only called in full language mode.
         The engine would have to be updated to call validation attributes before performing the `[ValidateTrustedData]` operation.

1. Allow Cmdlet to flow tainted flag

   * Some cmdlets may not have a reasonable way of validating the input,
      and may need to flow the fact that the data is tainted to the output object(s).

      This should be done via an attribute on the cmdlet.

1. Environment variables

   * As environment variable can be set by the user and in a constrained language mode you are protecting against the user,
      environment variables should be treated like untrusted input.

      **Issue:**  Strings returned from the environment provider, `$env:` and `[Environment]::GetEnvironmentVariable` are different objects every call.
      This means they will have to be tracked differently from other objects.

      * Proposed solution

         Track the environment variable in a different table by their hash.  `GetHashCode` should be sufficient as it should be the same inside the same process.

### Issues

#### Cached value types

Integer `0`-`100` are cached by the compiler.
This means `1` is always the same object and makes the current implementation not able to track these values.

## Alternate Proposals and Considerations

## PowerShell Committee Decision

### Voting Results

### Majority Decision

### Minority Decision

N/A
