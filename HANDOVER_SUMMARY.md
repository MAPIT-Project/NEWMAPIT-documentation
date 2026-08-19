# Business handover summary MAPIT

**Status date:** 20 July 2026

**Advice:** suitable for handover as a development project, **do not release as a production-ready service**

## Project description

MAPIT is a digital assistant for finding and assessing European funding opportunities and for
support in writing proposals. The product consists of a web application and an
AI-supported backend service. Horizon Europe is the most fully developed; the coverage and
validation of other European programmes is more limited.

## Handover status

After the review, the documentation is considerably more transferable: there is now one starting
point, a controlled overview of the interface between front and back end, and central information
on security, deployment, operations, data and known risks. Automatic checks have also been
added for both application parts, and a blockage in backend test collection has been resolved.

The original [documentation review](DOCUMENTATION_REVIEW.md) gave a readiness score of
**37/100**. That was a baseline measurement before the improvements and has not been formally
re-scored. The priority P0 documentation actions have been carried out, but the underlying
product risks have not been resolved by documentation alone. The most recent functional quality
measurement present remains **23 of 42 scenarios passed (54.8%)**, against a required 80%. The
current conclusion is therefore: transferable for further development and a controlled
demonstration environment, but not ready for unattended production or an independent release.

## Five most important risks

1. **Insufficiently proven product quality.** Nineteen of the 42 core scenarios fail in the
   latest final measurement. Important answers and conversation flows must be retested and
   improved before release (`QUAL-001`).
2. **A complete data environment is not reproducible.** The representative database and the
   required raw data are not fully delivered with the project. As a result, a new team cannot
   independently build the same environment (`DATA-001`, `DATA-002`).
3. **Access protection is not enforced everywhere.** User identity and access to shared or
   personal data are insufficiently controlled in parts of the chain. This can lead to
   unwanted viewing or modification (`API-001`, `SEC-006`, `SEC-007`).
4. **Security settings require recovery before production.** There are risks around a
   default secret key, session cookies, key naming and disabled protection against
   excessive usage (`SEC-002` through `SEC-005`).
5. **Continuity and operations are not yet arranged.** Backups, restore tests, alerts,
   responsibilities and automatic data refresh are described but not yet
   operationally demonstrated (`OPS-001`, `OPS-002`).

The full substantiation and workarounds are in [Known issues and risks](KNOWN_ISSUES_AND_RISKS.md).

## Carried out during the review

- A full documentation review with score, contradictions and prioritised actions.
- Nine new or fully revised P0 documents for onboarding, data, API, database,
  security, deployment, operations and risks; eight misleading documents have been marked
  as archive.
- Automatic checks for backend and frontend, plus recovery of the backend test collection and
  a recorded frontend error baseline.
- A migration overview with files, checksums, dependencies and known gaps.
- In Wave 3: complete overviews of environment variables, tools and dependencies, this
  business summary and a renewed documentation index.

## Next steps for the receiving party

1. Appoint one product owner and a technically responsible person per application part.
2. Formally accept the current status as a development project and do not plan a production
   release as long as the critical and high risks have not demonstrably been closed.
3. Have the five risks above resolved, with owner, end date and proof of control per item.
4. Deliver an approved, complete data set with version, origin and checksum, and test the
   build-up of a completely new environment.
5. Make all automatic quality checks green and blocking; then repeat the 42
   core scenarios and record the release decision.
6. Before going into use, carry out an independent security audit and a backup/restore test.

## Contact for questions

Technical handover questions: **Anmar Ram** — [anmar.noah@gmail.com](mailto:anmar.noah@gmail.com).

The organisation still has to appoint a permanent product owner, operations contact and escalation contact.
