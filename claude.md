# CLAUDE.md - WeWeb Component Development Guide

This file provides comprehensive guidance for developing WeWeb custom components in this repository.

## Development Commands

- **Install dependencies**: `npm i`
- **Serve locally**: `npm run serve --port=[PORT]` (then add custom element in WeWeb editor developer popup)
- **Build for release**: `npm run build --name=my-element` (check for build errors before release)

## WeWeb Component Architecture

This is a WeWeb custom element component built with Vue.js and configured using the WeWeb CLI framework.

### Project Structure
- `src/wwElement.vue` - Main Vue component with template, script, and scoped SCSS styling
- `ww-config.js` - WeWeb element configuration defining editor properties and settings
- `package.json` - Contains WeWeb CLI dependency and build/serve scripts

### Component Architecture
- **Props**: Component receives `content` object prop containing all configurable properties
- **Configuration**: Element properties are defined in `ww-config.js` and accessed via `props.content.propertyName`
- **Styling**: Uses scoped SCSS with Vue single-file component structure
- **WeWeb Integration**: Element integrates with WeWeb editor through configuration schema

## CRITICAL REQUIREMENTS

**THESE ARE MANDATORY AND MUST BE FOLLOWED IN ALL WeWeb COMPONENTS:**

- **MANDATORY & CRITICAL**: Use optional chaining (?.) for all content references
- **MANDATORY & CRITICAL**: All variable references, content properties, computed values, functions and any accessed data must include type safety checks (using optional chaining, nullish coalescing, type guards etc.) to prevent component crashes when values are undefined/null or of incorrect type - especially for props.content references which may not exist initially
- **MANDATORY & CRITICAL**: Every content properties must be considered reactive, when the user change their value in the editor the component must update in realtime
- **MANDATORY**: INCLUDE ALL USEFUL TRIGGERS AND INTERNAL VARIABLES
- **CRITICAL** Always import any external functions/utilities you plan to use
  - If using date-fns functions like addDays, import them: `import { addDays } from 'date-fns'`
  - If using lodash functions, import them: `import { get } from 'lodash'`
  - If using custom utilities, import them from their correct path
- Add ANY triggers that could be useful for NoCode users
- Add ANY internal variables that could be useful for NoCode users
- Think from a NoCode user perspective when adding these

## ABSOLUTELY CRITICAL TECHNICAL REQUIREMENTS

### Editor Code Blocks (MANDATORY):
- `/* wwEditor:start */` and `/* wwEditor:end */` blocks MUST be present in BOTH component code AND ww-config.js
- Required for ALL bindingValidation and propertyHelp
- EVERY wwEditor:start MUST have matching wwEditor:end
- Mismatched tags will cause catastrophic component failure

### Global Object Access (MANDATORY):
- NEVER access document/window directly
- ALWAYS use wwLib.getFrontDocument() for document
- ALWAYS use wwLib.getFrontWindow() for window
- Direct access breaks component isolation

### Component Root Element Sizing (CRITICAL):
- NEVER hardcode width/height on the root element
- Root element MUST fluidly adapt to user-defined dimensions
- Fixed dimensions on root element prevent proper NoCode customization
- Inner elements may have fixed dimensions as needed

### Array Property Requirements (ABSOLUTELY CRITICAL):
- **ALL Array properties containing objects MUST follow the WeWeb professional standard**
- **MANDATORY Array Structure in ww-config.js:**
  ```javascript
  arrayProperty: {
    label: { en: 'Items' },
    type: 'Array',
    section: 'settings',
    bindable: true,
    defaultValue: [
      { id: 'item1', name: 'Sample Item', value: 'data' }
    ],
    options: {
      expandable: true,
      getItemLabel(item) {
        return item.name || item.label || item.title || `Item ${item.id || 'Unknown'}`;
      },
      item: {
        type: 'Object',
        defaultValue: { id: 'item1', name: 'New Item', value: '' },
        options: {
          item: {
            id: { label: { en: 'ID' }, type: 'Text' },
            name: { label: { en: 'Name' }, type: 'Text' },
            value: { label: { en: 'Value' }, type: 'Text' }
          }
        }
      }
    },
    /* wwEditor:start */
    bindingValidation: {
      type: 'array',
      tooltip: 'Array of objects with required properties'
    },
    /* wwEditor:end */
  }
  ```

- **MANDATORY Formula Properties for Dynamic Field Mapping:**
  ```javascript
  arrayPropertyIdFormula: {
    label: { en: 'ID Field' },
    type: 'Formula',
    section: 'settings',
    options: content => ({
      template: Array.isArray(content.arrayProperty) && content.arrayProperty.length > 0 ? content.arrayProperty[0] : null,
    }),
    defaultValue: { type: 'f', code: "context.mapping?.['id']" },
    hidden: (content, sidepanelContent, boundProps) =>
      !Array.isArray(content.arrayProperty) || !content.arrayProperty?.length || !boundProps.arrayProperty,
  }
  ```

- **MANDATORY Vue Component Processing Pattern:**
  ```javascript
  const processedItems = computed(() => {
    const items = props.content?.arrayProperty || []
    const { resolveMappingFormula } = wwLib.wwFormula.useFormula()
    return items.map(item => {
      const id = resolveMappingFormula(props.content?.arrayPropertyIdFormula, item) ?? item.id
      const name = resolveMappingFormula(props.content?.arrayPropertyNameFormula, item) ?? item.name
      return { id: id || `item-${Date.now()}-${Math.random()}`, name: name || 'Untitled', originalItem: item, ...item }
    })
  })
  ```

### Select/Input Components (MANDATORY):
- MUST HAVE an initialValue property to bind the initial value
- MUST EXPOSE an internal variable using `wwLib.wwVariable.useComponentVariable`
- When initialValue changes, reset the internal variable
- When value changes, emit trigger event (AVOID INFINITE LOOPS)

### Component Reactivity (ABSOLUTELY CRITICAL):
- ALL props.content properties MUST be fully reactive
- NEVER use ref() or reactive() for data derived from props - use computed() instead
- NEVER use manual watchers for prop changes - computed properties handle this automatically

### Build Configuration (ABSOLUTELY CRITICAL):
- NO build configuration files: webpack.config.js, vite.config.js, rollup.config.js, .babelrc, tsconfig.json
- NO build dependencies in package.json: webpack, vite, babel, loaders
- Build process handled entirely by @weweb/cli
- Only include @weweb/cli as devDependency with "latest" version

### Package.json Requirements (CRITICAL):
- Name MUST NOT include "ww" or "weweb"
- Use specific versions for production dependencies (NOT "latest")

## WeWeb Development Patterns & Best Practices

### Component Props Structure
```javascript
export default {
  props: {
    uid: { type: String, required: true },
    content: { type: Object, required: true },
    /* wwEditor:start */
    wwEditorState: { type: Object, required: true },
    /* wwEditor:end */
  }
}
```

### TextSelect Properties (MANDATORY FORMAT)
```javascript
mySelect: {
  label: { en: 'Select Option' },
  type: 'TextSelect',
  section: 'settings',
  options: {
    options: [
      { value: 'value1', label: 'Label 1' },
      { value: 'value2', label: 'Label 2' },
    ]
  },
  defaultValue: 'value1',
  bindable: true,
  /* wwEditor:start */
  bindingValidation: { type: 'string', tooltip: 'Valid values: value1 | value2' },
  /* wwEditor:end */
}
```

### Internal Variables Pattern (MANDATORY for Interactive Components)
```javascript
const { value: internalValue, setValue: setInternalValue } = wwLib.wwVariable.useComponentVariable({
  uid: props.uid,
  name: 'value',
  type: 'string',
  defaultValue: 'my internal variable',
});
watch(() => props.content?.initialValue, (newValue) => {
  if (newValue !== undefined) setInternalValue(newValue);
}, { immediate: true });
const handleValueChange = (newValue) => {
  if (internalValue.value !== newValue) {
    setInternalValue(newValue);
    emit('trigger-event', { name: 'value-change', event: { value: newValue } });
  }
};
```

### CSS Variables for Dynamic Styling
```javascript
const dynamicStyles = computed(() => ({
  '--animation-speed': props.content?.animationSpeed || 1,
  '--primary-color': props.content?.primaryColor || '#ffffff',
}));
```

### Dropzone Implementation
```javascript
// ww-config.js
dropzoneContent: {
  hidden: true,
  defaultValue: [],
  /* wwEditor:start */
  bindingValidation: { type: 'array', tooltip: 'Array of elements to display in dropzone' },
  /* wwEditor:end */
}
```
```vue
<!-- wwElement.vue -->
<wwLayout path="dropzoneContent" direction="row" class="dropzone-container" />
```

### Event Handling
```javascript
// ww-config.js
triggerEvents: [
  { name: 'click', label: { en: 'On click' }, event: { value: '' } },
]
// wwElement.vue
emit('trigger-event', { name: 'click', event: { value: data } });
```

## Hipped-Specific Rules

- **DO NOT touch existing styling or style-related props** on forked components — WeWeb's native style panel handles all visual styling
- **Only add behaviour** — loading states, disabled states, success states, custom events
- All new props must be in the `settings` section of ww-config.js
- Spinner and state icons must use `currentColor` to inherit text colour automatically
- Naming convention for forked components: `hipped-[original-name]` (e.g. `hipped-ww-button`)

## hipped-ww-stack specifics

This fork exists solely as the companion element for `hipped-ww-kanban` (Hipped's recruitment platform: endorsements/candidate pipeline). `ww-kanban` renders one `ww-stack` instance per stack (and per stack×lane cell once swimlanes are on), passing configuration through as `ww-props` rather than `content.*` directly (`wwElementState.props` takes precedence over `content` throughout this component's computed properties).

`ww-kanban`'s `onChange` handler forwards its own `ww-props` back up (`this.customHandler(change, { ...this.wwElementState.props, updatedStackItems })`), so any new prop added here that `ww-kanban` needs downstream (e.g. `lane`) just needs to be included in the `ww-props` object `ww-kanban` passes in — no extra plumbing required on this side.

British English in all UI-facing labels, comments, and docs. No comparison language ("v2 adds X" / "unchanged from stock") in specs — state the current design as-is.
