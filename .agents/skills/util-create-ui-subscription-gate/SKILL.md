---
name: util-create-ui-subscription-gate
description: This skill walks you through implementing frontend subscription locks, rendering lock indicators 🔒, routing guards, and building attractive Upgrade overlays to drive product tier upsells.
---

# Skill: Implement Subscription Gates & Upgrade UI Panels

This skill guides you through implementing premium feature limits in the React/Nx monorepo. Instead of hiding premium modules entirely, you will render beautiful lock indicators 🔒 and professional "Upgrade Plan" overlay panels to offer an interactive trials experience.

---

## Design Golden Rules
1. **Interactive Upsell Experience**: NEVER completely hide premium modules or sub-features from navigation or menu trees. Show them with a premium lock 🔒 indicator.
2. **Harmonious Transitions**: Clicking on a locked module or action must smoothly trigger a premium styled "Upgrade Plan" overlay instead of throwing errors or rendering blank screens.
3. **Rich Aesthetics**: Upgrade overlays must look premium and high-fidelity, featuring harmonious color palettes (`var(--ui-primary)`), glassmorphism (backdrop filters), smooth micro-animations, and styled cards.
4. **Money Rule Precision**: Display price tags or subscription differences with proper currency symbols and neat formatting.

---

## Steps

### Step 1: Read the Licensed Feature Schema via Dynamic Decoupling

To prevent Nx monorepo boundary violations, the shared UI library (`@bes/shared-ui`) must remain decoupled from the main Shell application's Zustand store state. We resolve this by exposing a global registry checker in the library layer that is dynamically wired up by the Shell app on boot.

1. **Implement the Decoupled Global Registry Hook** inside `@bes/shared-ui` (`libs/shared-ui/src/lib/rbac.ts`):

```typescript
let isModuleLicensedFn: (moduleName: string) => boolean = () => true;

/**
 * Registers the global licensing evaluation callback.
 * Typically invoked by the main Shell application on startup.
 */
export function setLicenseChecker(checker: (moduleName: string) => boolean) {
  isModuleLicensedFn = checker;
}

/**
 * React hook to verify if a specific extension module is licensed.
 */
export function useFeatureLicense(moduleName: string): boolean {
  return isModuleLicensedFn(moduleName.toLowerCase());
}
```

2. **Wire Up the State Binding** inside the Shell application startup hook (`apps/shell/src/app/use-shell.ts`):

```typescript
import { useEffect } from 'react';
import { useAuthStore } from '../store/auth-store';
import { setLicenseChecker } from '@bes/shared-ui';

// Inside useShell hook
useEffect(() => {
  setLicenseChecker((moduleName: string) => {
    const active = useAuthStore.getState().activeModules;
    return active.includes(moduleName) || moduleName === 'settings' || moduleName === 'home';
  });
}, []);
```

---

### Step 2: Render Lock Indicators 🔒 in Navigation

Update sidebar navigation configuration or component lists to render a subtle lock icon next to unlicensed premium links:

```tsx
import React from 'react';
import { Lock } from 'lucide-react';
import { useFeatureLicense } from '@bes/shared-ui';

interface NavItemProps {
  label: string;
  moduleName: string;
  onClick: () => void;
}

export const SidebarItem: React.FC<NavItemProps> = ({ label, moduleName, onClick }) => {
  const isLicensed = useFeatureLicense(moduleName);

  return (
    <div 
      className={`nav-item ${!isLicensed ? 'locked' : ''}`}
      onClick={onClick}
      style={{
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'space-between',
        padding: '12px var(--ui-spacing-md)',
        cursor: 'pointer',
        transition: 'all 0.2s ease',
        opacity: isLicensed ? 1 : 0.7
      }}
    >
      <span>{label}</span>
      {!isLicensed && (
        <Lock 
          size={14} 
          style={{ 
            color: 'var(--ui-warning-color, #f59e0b)',
            marginLeft: '8px'
          }} 
        />
      )}
    </div>
  );
};
```

---

### Step 3: Implement the Upgrade Plan Overlay (Upsell Component)

Create a stunning, modern component inside `@bes/shared-ui` or in the Shell library that handles locked feature clicks:

```tsx
// libs/shared-ui/src/lib/UpgradeGateOverlay.tsx
import React from 'react';
import { Sparkles, ArrowRight } from 'lucide-react';
import { Button } from './button'; // Standard Shared-UI component

interface UpgradeGateOverlayProps {
  moduleName: string;
  requiredTier: 'Pro' | 'Premium';
  onClose?: () => void;
}

export const UpgradeGateOverlay: React.FC<UpgradeGateOverlayProps> = ({ moduleName, requiredTier, onClose }) => {
  return (
    <div style={{
      position: 'absolute',
      top: 0,
      left: 0,
      right: 0,
      bottom: 0,
      background: 'rgba(255, 255, 255, 0.85)',
      backdropFilter: 'blur(8px)',
      display: 'flex',
      alignItems: 'center',
      justifyContent: 'center',
      borderRadius: 'var(--ui-radius-lg)',
      zIndex: 100,
      padding: 'var(--ui-spacing-xl)',
      animation: 'fadeIn 0.3s cubic-bezier(0.16, 1, 0.3, 1)'
    }}>
      <div style={{
        background: 'var(--ui-white)',
        borderRadius: 'var(--ui-radius-xl)',
        border: '1px solid var(--ui-gray-200)',
        boxShadow: 'var(--ui-shadow-xl, 0 20px 25px -5px rgba(0,0,0,0.1))',
        padding: 'var(--ui-spacing-xl)',
        maxWidth: '440px',
        textAlign: 'center',
      }}>
        <div style={{
          display: 'inline-flex',
          padding: '12px',
          borderRadius: '50%',
          background: 'linear-gradient(135deg, #e0e7ff, #c7d2fe)',
          marginBottom: 'var(--ui-spacing-md)',
          color: 'var(--ui-primary)'
        }}>
          <Sparkles size={28} />
        </div>
        
        <h3 style={{ fontSize: 'var(--ui-text-lg)', fontWeight: 700, margin: '0 0 8px 0' }}>
          Unlock {moduleName.toUpperCase()} features
        </h3>
        
        <p style={{ fontSize: 'var(--ui-text-sm)', color: 'var(--ui-gray-600)', margin: '0 0 24px 0', lineHeight: 1.5 }}>
          This capability is included in the premium <strong>{requiredTier} Plan</strong>. Upgrading gives you access to advanced analytics, automated reports, and integrations.
        </p>

        <div style={{ display: 'flex', gap: '12px', justifyContent: 'center' }}>
          {onClose && (
            <Button variant="secondary" onClick={onClose}>
              Dismiss
            </Button>
          )}
          <Button variant="primary" style={{ display: 'inline-flex', alignItems: 'center' }}>
            Upgrade Now <ArrowRight size={16} style={{ marginLeft: '8px' }} />
          </Button>
        </div>
      </div>
    </div>
  );
};
```

---

### Step 4: Gracefully Protect Premium Views (View Wrapper)

Wrap premium page modules or sub-panels using the Gate Wrapper to overlay the upgrade card dynamically:

```tsx
import React, { useState } from 'react';
import { useFeatureLicense } from '@bes/shared-ui';
import { UpgradeGateOverlay } from '@bes/shared-ui';

export const AdvancedReportingPanel: React.FC = () => {
  const isLicensed = useFeatureLicense('finance');

  return (
    <div style={{ position: 'relative', minHeight: '350px', padding: 'var(--ui-spacing-lg)' }}>
      {/* 1. Show the Upsell Overlay if not licensed */}
      {!isLicensed && (
        <UpgradeGateOverlay 
          moduleName="Advanced Budget Forecasting & Reports" 
          requiredTier="Pro" 
        />
      )}

      {/* 2. Standard page layout operates normally in the background */}
      <h4 style={{ fontWeight: 600 }}>Budget Performance Forecasts</h4>
      <div className="forecast-chart-placeholder" style={{ filter: !isLicensed ? 'blur(2px)' : 'none' }}>
        {/* Real forecast stats ... */}
      </div>
    </div>
  );
};
```

---

### Step 5: Verify

1. Run the frontend application:
   ```bash
   npm run dev
   ```
2. Onboard a client with a **Basic** subscription plan.
3. Login as the client administrator and verify:
   - Sidebar renders lock 🔒 indicators on Pro/Premium features.
   - Clicking those sections redirects or overlays the beautiful **Upgrade Gate Overlay**.
   - No crash loops or error boundaries are triggered.
