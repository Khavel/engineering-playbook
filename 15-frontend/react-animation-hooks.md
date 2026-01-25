# React Animation Hooks Pattern

Reusable pattern for entrance/exit animations with proper mounting/unmounting.

## The Problem

Animating component exit is tricky because:
1. Component unmounts immediately when `isVisible` becomes false
2. Exit animation never plays
3. Need to delay unmount until animation completes

## Solution: useEntranceAnimation Hook

```tsx
import { useState, useEffect, useCallback } from 'react';

export const ANIMATION_DURATION = {
  FAST: 150,
  NORMAL: 200,
  SLOW: 300,
} as const;

export const ANIMATION_CLASS = {
  SCALE_IN: 'animate-scaleIn',
  SCALE_OUT: 'animate-scaleOut',
  FADE_IN: 'animate-fadeIn',
  FADE_OUT: 'animate-fadeOut',
} as const;

export function useEntranceAnimation(
  isVisible: boolean,
  duration: number = ANIMATION_DURATION.NORMAL
) {
  const [shouldRender, setShouldRender] = useState(isVisible);
  const [animationClass, setAnimationClass] = useState('');

  useEffect(() => {
    if (isVisible) {
      // Show immediately, entrance animation handled by CSS
      setShouldRender(true);
      setAnimationClass('');
    } else if (shouldRender) {
      // Trigger exit animation
      setAnimationClass(ANIMATION_CLASS.SCALE_OUT);

      // Delay unmount until animation completes
      const timer = setTimeout(() => {
        setShouldRender(false);
        setAnimationClass('');
      }, duration);

      return () => clearTimeout(timer);
    }
  }, [isVisible, shouldRender, duration]);

  return { shouldRender, animationClass };
}
```

## Usage in Modal Component

```tsx
export const Modal: FC<ModalProps> = ({ isOpen, onClose, children }) => {
  const { shouldRender, animationClass } = useEntranceAnimation(isOpen);

  if (!shouldRender) return null;

  return (
    <div className={`modal-backdrop ${animationClass}`}>
      <div className={`modal-content ${animationClass}`}>
        {children}
      </div>
    </div>
  );
};
```

## CSS Animations

```css
@keyframes scaleIn {
  from { opacity: 0; transform: scale(0.95); }
  to { opacity: 1; transform: scale(1); }
}

@keyframes scaleOut {
  from { opacity: 1; transform: scale(1); }
  to { opacity: 0; transform: scale(0.95); }
}

.animate-scaleIn {
  animation: scaleIn 200ms ease-out forwards;
}

.animate-scaleOut {
  animation: scaleOut 150ms ease-in forwards;
}
```

## Reduced Motion Support

```tsx
export function getAnimationClass(
  className: string,
  prefersReducedMotion: boolean
): string {
  return prefersReducedMotion ? '' : className;
}

// Usage with media query hook
const prefersReducedMotion = useMediaQuery('(prefers-reduced-motion: reduce)');
const safeClass = getAnimationClass(ANIMATION_CLASS.SCALE_IN, prefersReducedMotion);
```

## Key Insights

1. **shouldRender vs isVisible**: Component stays mounted during exit animation
2. **Cleanup timers**: Always return cleanup function from useEffect
3. **CSS forwards**: Use `animation-fill-mode: forwards` to hold final state
4. **Exit faster than enter**: Exit animations feel snappier at ~75% of entrance duration
