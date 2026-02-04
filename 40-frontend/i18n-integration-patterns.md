# i18n Integration Patterns - React Context API

**Problem:** New developers often assume i18n libraries export a `useTranslation()` hook and use function-call syntax like `t('key.path')`, but custom implementations use different patterns.

**Solution:** Establish clear i18n architecture patterns upfront to prevent TypeScript errors and compilation failures.

## Pattern: Object-Based Translation Access

### Implementation
```typescript
// Context: lib/i18n/context.tsx
interface LanguageContextValue {
  language: Language;
  setLanguage: (lang: Language) => void;
  t: TranslationKeys;  // ← Entire translation object, not a function
}

export function useLanguage(): LanguageContextValue {
  const context = useContext(LanguageContext);
  if (!context) throw new Error('useLanguage must be used within LanguageProvider');
  return context;
}

// Usage in components
const { t } = useLanguage();
const message = t.section.key;  // ← Object notation, not function call
```

### Translation Structure
```typescript
// Translations are nested objects with 'as const' for type safety
export const en = {
  common: {
    loading: 'Loading...',
    error: 'Error',
  },
  arcEvents: {
    title: 'Active Events',
    noEvents: 'No events currently available.',
  },
} as const;

export type TranslationKeys = DeepStringify<typeof en>;
```

## Common Mistakes & Fixes

### ❌ Wrong: Function Call Pattern
```typescript
// This FAILS if t is an object
const { t } = useLanguage();
<h1>{t('common.title')}</h1>  // ← TypeError: t is not a function
```

### ✓ Correct: Object Access Pattern  
```typescript
const { t } = useLanguage();
<h1>{t.common.title}</h1>  // ← Works with object-based structure
```

## Integration Checklist for New Features

1. **Import the correct hook**
   ```typescript
   import { useLanguage } from '@/lib/i18n';  // ← Not useTranslation
   ```

2. **Destructure correctly**
   ```typescript
   const { t, language, setLanguage } = useLanguage();
   ```

3. **Access translations with dot notation**
   ```typescript
   const label = t.section.key;  // ← Not t('section.key')
   ```

4. **Ensure type safety** (IDE will autocomplete)
   ```typescript
   t.arcEvents.title    // ✓ Valid: TypeScript knows all valid paths
   t.invalidPath        // ✗ Error: TypeScript catches typos
   ```

5. **Test mocks match the pattern**
   ```typescript
   jest.mock('@/lib/i18n', () => ({
     useLanguage: () => ({
       t: { common: { title: 'Title' }, /* ... */ },
       language: 'en',
       setLanguage: jest.fn(),
     }),
   }));
   ```

## Why This Pattern Works

- **Type safety:** Entire translation structure is known to TypeScript
- **IDE autocomplete:** IDEs can suggest valid translation keys
- **Compile-time validation:** Typos caught before runtime
- **No runtime overhead:** No function calls or string parsing needed
- **Explicit structure:** Clear nesting matches UI organization

## Performance Notes

- Object access: O(1) - direct property lookup
- No string parsing: Faster than i18next/react-i18next function patterns
- Memoized in context: Re-renders only when language changes
- Suitable for components with many translation references

## When to Refactor Away

If managing 100+ translation keys becomes unwieldy:
- Consider structured comments/docs for organization
- Split translations into separate files and import
- Still maintain object-based access pattern (don't switch to function calls)

## Related Files
- RaidersRep: `web/src/lib/i18n/` (context, translations, type definitions)
- Example component: `web/src/components/arc-events/ArcEventTimersWidget.tsx`
