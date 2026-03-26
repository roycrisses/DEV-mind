---
title: Unit Test Suite Generator
category: testing
version: 1.0.0
works_with: [claude, openai, ollama, gemini]
---

## Purpose
Generate a comprehensive unit test suite for any function or class — covering the happy path, edge cases, and error conditions.

## When to Use
- Writing tests for existing code that has none
- After finishing a feature and want test coverage before PR
- When a bug was fixed and you want to add a regression test

## The Prompt

```
You are a senior {{LANGUAGE}} engineer and testing expert. Write a comprehensive unit test suite for the following code.

**Language / Framework:** {{LANGUAGE}}
**Test framework:** {{TEST_FRAMEWORK}}
(Examples: "Jest", "Pytest with pytest-mock", "Go testing package", "JUnit 5", "RSpec")

**Code to test:**
```{{LANGUAGE}}
{{CODE}}
```

**External dependencies (things to mock):**
{{DEPENDENCIES}}
(Examples: "database calls via UserRepository", "HTTP calls via axios", "filesystem via fs module", "none")

**Coverage target:** {{COVERAGE_TARGET}}
(Examples: "happy path only — just make it non-zero", "80% branch coverage", "exhaustive — every code path")

Generate:
1. **Test plan** — A list of all test cases BEFORE writing code (red-green-refactor discipline)
2. **Full test file** — Complete, runnable test code with:
   - Proper setup/teardown (beforeEach, afterEach)
   - All mocks configured correctly
   - Descriptive test names in "it should..." format
   - Grouped with describe() blocks
   - No test interdependence — each test is isolated
3. **Coverage analysis** — Which branches/paths are NOT covered and why
4. **How to run** — The exact command to run these tests
```

## Example Input

| Variable | Value |
|---|---|
| `{{LANGUAGE}}` | `TypeScript` |
| `{{TEST_FRAMEWORK}}` | `Jest` |
| `{{CODE}}` | A `UserService.createUser()` method that validates input, hashes a password, checks for email conflicts, and inserts to DB |
| `{{DEPENDENCIES}}` | `UserRepository (database), bcrypt (password hashing), EmailService (sends welcome email)` |
| `{{COVERAGE_TARGET}}` | `exhaustive — every code path` |

## Example Output

```typescript
// user-service.test.ts
import { UserService } from './user-service';
import { UserRepository } from './user-repository';
import { EmailService } from './email-service';
import bcrypt from 'bcrypt';

jest.mock('./user-repository');
jest.mock('./email-service');
jest.mock('bcrypt');

describe('UserService.createUser()', () => {
  let service: UserService;
  let mockRepo: jest.Mocked<UserRepository>;
  let mockEmail: jest.Mocked<EmailService>;

  beforeEach(() => {
    mockRepo = new UserRepository() as jest.Mocked<UserRepository>;
    mockEmail = new EmailService() as jest.Mocked<EmailService>;
    service = new UserService(mockRepo, mockEmail);
    (bcrypt.hash as jest.Mock).mockResolvedValue('hashed_password');
  });

  afterEach(() => jest.clearAllMocks());

  describe('when input is valid', () => {
    it('should hash the password before saving', async () => {
      mockRepo.findByEmail.mockResolvedValue(null);
      mockRepo.create.mockResolvedValue({ id: '1', email: 'a@b.com' });
      await service.createUser({ email: 'a@b.com', password: 'secret123' });
      expect(bcrypt.hash).toHaveBeenCalledWith('secret123', 10);
      expect(mockRepo.create).toHaveBeenCalledWith(
        expect.objectContaining({ password: 'hashed_password' })
      );
    });

    it('should send a welcome email after creation', async () => {
      mockRepo.findByEmail.mockResolvedValue(null);
      mockRepo.create.mockResolvedValue({ id: '1', email: 'a@b.com' });
      await service.createUser({ email: 'a@b.com', password: 'secret123' });
      expect(mockEmail.sendWelcome).toHaveBeenCalledWith('a@b.com');
    });

    it('should return the created user without the password field', async () => {
      mockRepo.findByEmail.mockResolvedValue(null);
      mockRepo.create.mockResolvedValue({ id: '1', email: 'a@b.com' });
      const result = await service.createUser({ email: 'a@b.com', password: 'secret123' });
      expect(result).not.toHaveProperty('password');
    });
  });

  describe('when email is already taken', () => {
    it('should throw a ConflictError without calling create', async () => {
      mockRepo.findByEmail.mockResolvedValue({ id: '99', email: 'a@b.com' });
      await expect(
        service.createUser({ email: 'a@b.com', password: 'secret123' })
      ).rejects.toThrow('Email already in use');
      expect(mockRepo.create).not.toHaveBeenCalled();
    });
  });

  describe('when input is invalid', () => {
    it('should throw ValidationError for missing email', async () => {
      await expect(
        service.createUser({ email: '', password: 'secret123' })
      ).rejects.toThrow('Email is required');
    });

    it('should throw ValidationError when password is under 8 characters', async () => {
      await expect(
        service.createUser({ email: 'a@b.com', password: 'short' })
      ).rejects.toThrow('Password must be at least 8 characters');
    });
  });

  describe('when database fails', () => {
    it('should propagate the database error and NOT send a welcome email', async () => {
      mockRepo.findByEmail.mockResolvedValue(null);
      mockRepo.create.mockRejectedValue(new Error('DB connection lost'));
      await expect(
        service.createUser({ email: 'a@b.com', password: 'secret123' })
      ).rejects.toThrow('DB connection lost');
      expect(mockEmail.sendWelcome).not.toHaveBeenCalled();
    });
  });
});
```

**Run tests:** `npx jest user-service.test.ts --coverage`

**Uncovered:** Email service failure path (if `sendWelcome` throws) — add this if email
delivery is critical to your business logic.
```

## Tips
- List all `{{DEPENDENCIES}}` explicitly  — if you miss one, the tests will try to make real DB/HTTP calls
- Use `{{COVERAGE_TARGET}}: "happy path only"` when you need tests fast (CI gate); use `"exhaustive"` before releasing critical features
- Ask the AI to add a `TODO: add integration test for X` comment for paths it intentionally doesn't unit-test — keeps the coverage gaps visible
