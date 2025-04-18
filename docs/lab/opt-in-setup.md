# Making Ollama Lab Features Opt-in

This document outlines the steps required to make the Ollama lab features (model management, running models view, etc.) an opt-in feature that can be controlled through admin settings.

## Implementation Steps

### 1. Backend Configuration

Add a new configuration in `backend/open_webui/config.py`:

```python
ENABLE_OLLAMA_LAB = PersistentConfig(
    "ENABLE_OLLAMA_LAB",
    "lab.ollama.enable",
    os.getenv("ENABLE_OLLAMA_LAB", "False").lower() == "true",
)
```

### 2. User Permissions

Add new permission settings in `backend/open_webui/config.py`:

```python
# Add with other USER_PERMISSIONS_FEATURES
USER_PERMISSIONS_FEATURES_OLLAMA_LAB = (
    os.environ.get("USER_PERMISSIONS_FEATURES_OLLAMA_LAB", "True").lower() == "true"
)

# Add to the permissions dictionary
"ollama_lab": USER_PERMISSIONS_FEATURES_OLLAMA_LAB,
```

### 3. Admin Settings UI

Create or update the Lab settings component at `src/lib/components/admin/Settings/Lab.svelte`:

```svelte
<div class="mb-3">
	<div class="mb-2.5 flex w-full justify-between">
		<div class="self-center text-xs font-medium">
			{$i18n.t('Ollama Lab')}
			<Tooltip>
				{$i18n.t('Enable advanced Ollama model management features')}
			</Tooltip>
		</div>
		<div class="flex items-center relative">
			<Switch bind:state={labConfig.ENABLE_OLLAMA_LAB} />
		</div>
	</div>
</div>
```

### 4. Route Protection

Update the route guard to check for the feature flag:

```typescript
// In the route layout or guard
if (!$settings?.enable_ollama_lab) {
	throw redirect(302, '/lab');
}
```

### 5. Environment Variables

Add the new environment variable to docker-compose files:

```yaml
ENABLE_OLLAMA_LAB: 'false' # disabled by default
```

### 6. API Endpoint Protection

Add checks in relevant API endpoints:

```python
if not app.state.config.ENABLE_OLLAMA_LAB:
    raise HTTPException(status_code=403, detail="Ollama Lab is not enabled")
```

## Notes

- The feature will be disabled by default and can be enabled through environment variables
- Admins can control the feature globally through the admin settings
- The feature can be controlled per-user through permissions
- The UI will adapt to show/hide the feature based on these settings
- Implementation follows the same pattern as web search and image generation features

## Implementation Timing

These changes can be implemented at any time without affecting the core functionality. The opt-in system is:

- Additive (no refactoring required)
- Modular (follows existing patterns)
- Isolated (changes are contained to specific areas)
- Non-breaking (existing code won't be affected)

This means development of Ollama lab features can continue independently of when these opt-in controls are implemented.
