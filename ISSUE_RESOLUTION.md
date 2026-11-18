# Issue Resolution: `URLSafeSerializer.loads` silently ignores unexpected arguments (such as `max_age`)

## 📌 Metadata & Scope
* **Repository:** [pallets/itsdangerous](https://github.com/pallets/itsdangerous)
* **Upstream Issue:** [Issue #429](https://github.com/pallets/itsdangerous/issues/429)
* **Status:** ✅ Resolved & Verified
* **Component:** `itsdangerous.serializer.Serializer.loads`
* **Affected File:** `src/itsdangerous/serializer.py`
* **Test File:** `tests/test_itsdangerous/test_url_safe.py`

---

## 🔍 Problem Description

In `itsdangerous`, `TimedSerializer` and `URLSafeTimedSerializer` implement timestamp validation and require/accept a `max_age` argument to expire tokens after a duration.

However, untimed serializers (like `Serializer` and `URLSafeSerializer`) defined:
```python
def loads(self, s: str | bytes, salt: str | bytes | None = None, **kwargs: t.Any) -> t.Any:
```
and completely ignored `**kwargs` without checking or validating them.

### Security Impact
If a developer accidentally changes from `URLSafeTimedSerializer` to `URLSafeSerializer`, or incorrectly believes `URLSafeSerializer` checks `max_age=3600`, calls like:
```python
serializer.loads(signed, max_age=3600)
```
succeeded without warning, silently ignoring `max_age`. Expired tokens would be accepted indefinitely.

---

## 🛠️ Root Cause

`Serializer.loads` accepted `**kwargs` for signature compatibility, but did not guard against unexpected keyword arguments. Untimed serializers do not support time-based parameters like `max_age`.

---

## 🧪 Test Case Reproduction

Added `test_urlsafe_serializer_rejects_unexpected_kwargs` in [`tests/test_itsdangerous/test_url_safe.py`](file:///Users/apple/Desktop/jmd/itsdangerous/tests/test_itsdangerous/test_url_safe.py):

```python
def test_urlsafe_serializer_rejects_unexpected_kwargs():
    serializer = URLSafeSerializer("SECRET")
    signed = serializer.dumps("value")
    with pytest.raises(TypeError, match="unexpected keyword argument"):
        serializer.loads(signed, max_age=10)
```

Before the fix, running pytest failed with:
```bash
E Failed: DID NOT RAISE <class 'TypeError'>
tests/test_itsdangerous/test_url_safe.py:30: Failed
```

---

## Solution Approach
1. **Fail-Fast Security Principle:** Silently ignoring unexpected keyword arguments like `max_age` on `URLSafeSerializer.loads()` creates a dangerous security pitfall where developers believe signature expiration is enforced when it is not.
2. **Strict Signature Validation:** Instead of accepting and discarding arbitrary `**kwargs` in `Serializer.loads()`, enforce strict argument forwarding so unsupported kwargs (such as `max_age` on non-timed serializers) immediately raise a `TypeError` rather than being swallowed.
3. **TDD & Regression Safety:** Reproduced the silent-ignore bug with a failing test in `tests/test_itsdangerous/test_url_safe.py` and verified timed serializers (`URLSafeTimedSerializer`) continue accepting `max_age` cleanly.

## 💡 Solution

Updated `Serializer.loads` in `src/itsdangerous/serializer.py` to raise a descriptive `TypeError` when unexpected keyword arguments are passed:

```python
def loads(
    self, s: str | bytes, salt: str | bytes | None = None, **kwargs: t.Any
) -> t.Any:
    if kwargs:
        unexpected = next(iter(kwargs))
        raise TypeError(
            f"{type(self).__name__}.loads() got an unexpected"
            f" keyword argument {unexpected!r}"
        )

    s = want_bytes(s)
    ...
```

---

## ✅ Verification
* **Pytest:** `298 passed in 0.69s` (100% passing)
* **Linter (ruff):** `All checks passed!`
