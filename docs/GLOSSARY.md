# Glossary

Short forms are written out the first time they appear in each document. This page lists them in one place, in plain language.

| Short form | Full name | Plain meaning |
|---|---|---|
| API | Application programming interface | The set of web addresses the application offers to apps |
| AWS | Amazon Web Services | The cloud provider used as the reference deployment |
| AZ | Availability zone | A separate data-centre area in the same region. Two zones means one failure does not take everything down |
| CDN | Content delivery network | Servers close to the user that handle encryption and absorb junk traffic |
| CI/CD | Continuous integration and continuous delivery | Automatic checks on every change, then a controlled release |
| CPU | Central processing unit | The processor. "Processor-heavy" means the work uses a lot of computing time |
| CORS | Cross-origin resource sharing | Rules for which websites may call this application from a browser |
| CQRS | Command and query separation | A pattern that splits reads and writes. Not used here |
| CRUD | Create, read, update, and delete | The four basic actions on a record |
| CSRF | Cross-site request forgery | A trick that makes a browser send a request the person did not intend |
| DAU | Daily active users | People who use the product on a given day |
| DDL | Data definition language | Commands that change the shape of the database |
| DDoS | Distributed denial of service | A flood of traffic meant to knock the service offline |
| DNS | Domain Name System | Turns a name such as `api.example.com` into a network address |
| EdDSA | Edwards-curve Digital Signature Algorithm | The method used to sign login tokens so they cannot be forged |
| GDPR | General Data Protection Regulation | European rules for personal data, including the right to be deleted |
| GCP | Google Cloud Platform | An alternative cloud provider. The design can move there |
| HLD | High-Level Design | The overview of the main parts and how they connect |
| HSTS | HTTP Strict Transport Security | Tells browsers to use an encrypted connection only |
| HTTP | Hypertext Transfer Protocol | The language browsers and apps use to talk to a server |
| IAM | Identity and access management | Who is allowed to touch cloud resources |
| IP | Internet Protocol address | The network address of a computer |
| JSON | JavaScript Object Notation | The text format used for requests and responses |
| JWT | JSON Web Token | A signed pass that proves who the caller is, without a database lookup |
| JWKS | JSON Web Key Set | The published list of keys used to check those passes |
| KMS | Key Management Service | A locked store for encryption keys |
| MFA | Multi-factor authentication | A second proof at login, such as a phone code. Planned before launch for administrators |
| NAT | Network Address Translation | The only door from private servers out to the internet |
| OIDC | OpenID Connect | A standard way to sign in with a company identity provider |
| ORM | Object-relational mapping | The library that turns database rows into Python objects |
| OWASP | Open Worldwide Application Security Project | The group that publishes the main list of web application risks |
| PII | Personally identifiable information | Data that identifies a person, such as an email address |
| PITR | Point-in-time recovery | Restore the database to the minute before something went wrong |
| REST | Representational State Transfer | The style of web interface used here: nouns, standard methods, and status codes |
| RPO | Recovery point objective | The most data we can afford to lose. Here, five minutes |
| RTO | Recovery time objective | The longest we can take to restore service. Here, thirty minutes |
| SBOM | Software bill of materials | A list of every library shipped in the application |
| SCA | Software composition analysis | A scan of those libraries for known flaws |
| SLO | Service level objective | A promised level of speed or uptime |
| SPA | Single-page application | A website that loads once and then updates in place |
| SPOF | Single point of failure | One part whose failure stops the whole system |
| SQL | Structured Query Language | The language used to read and write the database |
| SSO | Single sign-on | Signing in once through a company account |
| TLS | Transport Layer Security | Encryption for data moving over the network |
| TTL | Time to live | How long a cached value is kept |
| URL | Uniform Resource Locator | A web address |
| UUID | Universally unique identifier | An identifier that is very hard to guess |
| VPC | Virtual private cloud | A private network that the public internet cannot enter |
| WAF | Web application firewall | A filter that blocks common attacks before they reach the application |
| WAL | Write-ahead log | The database journal used to recover recent changes |
| XSS | Cross-site scripting | Injecting a script into a page another person will open |
