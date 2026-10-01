# OcéEns

EPF course evaluation platform: surveys are created by study track, students complete them, and the responses are exported, visualized, and summarized.

## Language

### Authentication

**Development login** (`AUTH_MODE=dev`):
Login without an identity provider: you select a user's email address and log in as that user, without proof of identity. This mode exists only when `AUTH_MODE=dev` and must never be used in production.
_Avoid_: impersonation, identity theft, fake login.

### French Vocabulary

**sondage**: It means "summary" in English. It should not be translate to English in the whole project

**synthèse**: It means "survey" in English. It should not be translate to English in the whole project