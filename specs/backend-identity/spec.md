# backend-identity Specification

## Purpose

Owns player identity for Air-Pilote: registration, login, JWT access and refresh token issuance, refresh token rotation with reuse/family detection, and logout/revocation. Passwords are hashed behind a port (Argon2id). Refresh tokens are delivered as httpOnly Secure SameSite=Strict cookies; access tokens are delivered in the Authorization header AND/OR an httpOnly cookie.

## Requirements

### Requirement: Registration

The system MUST register a new player using a unique email and a password that satisfies a strength policy, MUST hash the password via the password hasher port, MUST reject duplicate emails, and SHALL return access and refresh tokens on success.

#### Scenario: Successful registration

- GIVEN no player exists with email `a@b.co`
- WHEN a registration request with a compliant password is submitted
- THEN a player account is created, the password is stored hashed (never plaintext), and access + refresh tokens are issued

#### Scenario: Duplicate email rejected

- GIVEN a player already exists with email `a@b.co`
- WHEN a registration request with the same email is submitted
- THEN the request is rejected with a duplicate-email error and no tokens are issued

#### Scenario: Weak password rejected

- GIVEN a registration request is submitted
- WHEN the password fails the strength policy
- THEN the request is rejected with a password-strength error and no account is created

### Requirement: Login

The system MUST verify credentials on login, MUST reject invalid credentials, and SHALL issue access and refresh tokens on success.

#### Scenario: Successful login

- GIVEN a player exists with email `a@b.co` and a known password
- WHEN login is submitted with matching credentials
- THEN access and refresh tokens are issued

#### Scenario: Invalid credentials

- GIVEN a player exists
- WHEN login is submitted with a wrong password
- THEN the request is rejected with a generic invalid-credentials error and no tokens are issued

### Requirement: Refresh Token Rotation

The system MUST rotate refresh tokens on each refresh, MUST store refresh tokens hashed, MUST deliver the new refresh token in an httpOnly Secure SameSite=Strict cookie, and SHALL detect reuse of an already-rotated refresh token by revoking the entire token family.

#### Scenario: Successful rotation

- GIVEN a valid, non-rotated refresh token is presented
- WHEN the refresh endpoint is called
- THEN a new access token and a new refresh token are issued, and the old refresh token is invalidated

#### Scenario: Reuse detected revokes family

- GIVEN refresh token R1 was already rotated into R2
- WHEN the now-invalid R1 is presented to the refresh endpoint
- THEN the entire token family (R1, R2, and all descendants) is revoked and no new tokens are issued

#### Scenario: Expired or invalid refresh rejected

- GIVEN a refresh token is expired or malformed
- WHEN the refresh endpoint is called
- THEN the request is rejected and no tokens are issued

### Requirement: Access Token Delivery

The system MUST issue a short-lived access token deliverable via the Authorization header AND/OR an httpOnly cookie.

#### Scenario: Access token usable for authenticated calls

- GIVEN a player holds a valid access token
- WHEN the player calls a protected endpoint presenting the access token
- THEN the request is authenticated and authorized for that player

#### Scenario: Expired access token

- GIVEN an access token has expired
- WHEN a protected endpoint is called
- THEN the request is rejected as unauthenticated and the client SHOULD attempt a refresh

### Requirement: Logout / Revocation

The system MUST revoke the presented refresh token family on logout and SHALL invalidate the refresh cookie.

#### Scenario: Logout revokes family

- GIVEN a player holds an active refresh token
- WHEN logout is called
- THEN the refresh token family is revoked and the refresh cookie is cleared

### Requirement: Password Hashing Port

The system MUST hash and verify passwords through a password hasher port (Argon2id) and MUST NOT store or log plaintext passwords.

#### Scenario: Plaintext never persisted

- GIVEN a registration or password change occurs
- WHEN the password is stored
- THEN only the Argon2id hash is persisted and the plaintext is discarded