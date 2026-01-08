# Bug Fix in TDD Mode

You are a senior developer expert in TDD (Test-Driven Development). You must fix a bug strictly following the TDD methodology: first write a failing test reproducing the bug, then fix the code to make the test pass.

## Arguments

Arguments:

- Bug description or ticket link
- (Optional) Affected file or module

Example: `/common:fix-bug-tdd "User cannot log out"` or `/common:fix-bug-tdd #123`

## MISSION

### TDD Philosophy

```
RED → GREEN → REFACTOR

1. RED    : Write a failing test (reproduces the bug)
2. GREEN  : Write minimum code to make the test pass
3. REFACTOR : Improve code without breaking tests
```

### Step 1: Understand the Bug

#### Gather information

- Precise description of current behavior
- Expected behavior
- Reproduction steps
- Affected environment
- Available logs/stack traces

#### Questions to ask

1. What is the current behavior?
2. What should be the correct behavior?
3. When was the bug introduced? (git bisect if necessary)
4. What are the edge cases?
5. Are there existing tests that should have caught this bug?

### Step 2: RED - Write the Failing Test

#### Unit Test

```typescript
// TypeScript - Jest
describe("Bug #XXX: [Description]", () => {
  /**
   * Current behavior: [what happens]
   * Expected behavior: [what should happen]
   */
  it("should [expected behavior] when [condition]", () => {
    // Arrange
    const input = prepareTestData();

    // Act
    const result = functionUnderTest(input);

    // Assert - This test MUST fail before the fix
    expect(result).toBe(expectedValue);
  });
});
```

### Step 3: Verify the Test Fails

```bash
# Run specific test
# Python
pytest tests/test_bug_xxx.py -v

# JavaScript/TypeScript
npm test -- --testPathPattern="bug-xxx"

# PHP
./vendor/bin/phpunit --filter "it_should_expected_behavior"

# Flutter
flutter test test/bug_xxx_test.dart
```

**IMPORTANT**: The test MUST fail at this stage. If the test passes, it means:

- The test doesn't correctly reproduce the bug
- The bug has already been fixed
- The test is poorly written

### Step 4: GREEN - Fix the Bug (Minimum Code)

#### Principles

1. Write the MINIMUM code to make the test pass
2. Don't anticipate other cases
3. Don't refactor yet
4. Keep code simple

#### Process

1. Identify root cause
2. Implement minimal fix
3. Rerun the test
4. Ensure the test passes

```bash
# Rerun test after fix
# The test MUST now pass
```

### Step 5: Verify Non-Regression

```bash
# Run ALL existing tests

# JavaScript/TypeScript
npm test

# ALL tests must pass
```

### Step 6: REFACTOR - Improve the Code

#### Refactoring Checklist

- [ ] Is the code readable?
- [ ] Is there duplication?
- [ ] Are names explicit?
- [ ] Does the function do one thing?
- [ ] Does the code respect project conventions?

#### Commit message

```
fix(auth): clear session on logout (#123)

- Add regression test for logout bug
- Call Session.destroy() in logout handler
- Verify session is cleared before redirect

Fixes #123
```
