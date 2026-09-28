# 03-components-a.md

File in questo blocco:

- `src/components/isa/app-shell.tsx`
- `src/components/isa/back-button.tsx`
- `src/components/isa/header.tsx`
- `src/components/isa/logo.tsx`
- `src/components/isa/mini-chart.tsx`
- `src/components/isa/module-picker-modal.tsx`
- `src/components/isa/new-solution-modal.tsx`
- `src/components/isa/share-modal.tsx`
- `src/components/isa/sidebar.tsx`
- `src/components/isa/solution-card.tsx`
- `src/components/isa/solution-row.tsx`
- `src/components/isa/ui/isa-menu.tsx`
- `src/components/isa/ui/isa-modal.tsx`

---

### `src/components/isa/app-shell.tsx`

106 righe

```tsx
import { createContext, useContext, useState } from "react";
import { useSolutions } from "@/lib/solutions-store";
import { BackButton } from "./back-button";
import { type CardView, IsaHeader } from "./header";
import { IsaSidebar } from "./sidebar";
import { NewSolutionModal } from "./new-solution-modal";

const NewSolutionContext = createContext<() => void>(() => {});
export const useNewSolution = () => useContext(NewSolutionContext);

export function BackgroundBlobs() {
  return (
    <div aria-hidden className="pointer-events-none fixed inset-0 -z-10">
      <div
        className="blob left-[-14%] top-[-16%] size-[520px]"
        style={{ background: "var(--blob-1)" }}
      />
      <div
        className="blob right-[-12%] top-[12%] size-[480px]"
        style={{ background: "var(--blob-2)" }}
      />
      <div
        className="blob bottom-[-20%] left-[26%] size-[560px]"
        style={{ background: "var(--blob-3)" }}
      />
    </div>
  );
}

export function AppShell({
  children,
  query,
  onQueryChange,
  view,
  onViewChange,
  back,
  backLabel,
}: {
  children: React.ReactNode;
  query?: string;
  onQueryChange?: (value: string) => void;
  view?: CardView;
  onViewChange?: (view: CardView) => void;
  back?: boolean;
  backLabel?: string;
}) {
  const [modalOpen, setModalOpen] = useState(false);
  const { createSolution } = useSolutions();

  return (
    <NewSolutionContext.Provider value={() => setModalOpen(true)}>
      <div className="relative min-h-screen overflow-hidden">
        <BackgroundBlobs />

        <div className="mx-auto flex max-w-[1600px] gap-4 p-4">
          <IsaSidebar />
          <div className="flex min-w-0 flex-1 flex-col gap-4">
            <IsaHeader
              query={query}
              onQueryChange={onQueryChange}
              view={view}
              onViewChange={onViewChange}
              onNewSolution={() => setModalOpen(true)}
            />
            {back && <BackButton label={backLabel ?? "Indietro"} className="self-start" />}
            <main className="min-w-0 flex-1">{children}</main>
          </div>
        </div>

        <NewSolutionModal
          open={modalOpen}
          onClose={() => setModalOpen(false)}
          onCreate={(input) => {
            createSolution(input);
            setModalOpen(false);
          }}
        />
      </div>
    </NewSolutionContext.Provider>
  );
}

export function PlaceholderPage({
  title,
  description,
  hint,
}: {
  title: string;
  description: string;
  hint?: string;
}) {
  return (
    <AppShell back backLabel="Indietro">
      <section className="glass-panel rounded-3xl p-8">
        <h2 className="text-2xl font-semibold tracking-tight">{title}</h2>
        <p className="mt-2 max-w-xl text-sm text-muted-foreground">{description}</p>
        {hint && (
          <p className="glass-chip mt-6 inline-block rounded-full px-3 py-1.5 text-xs text-muted-foreground">
            {hint}
          </p>
        )}
      </section>
    </AppShell>
  );
}
```

### `src/components/isa/back-button.tsx`

30 righe

```tsx
import { useRouter } from "@tanstack/react-router";
import { ArrowLeft } from "lucide-react";

export function BackButton({
  label = "Indietro",
  className = "",
}: {
  label?: string;
  className?: string;
}) {
  const router = useRouter();

  return (
    <button
      type="button"
      onClick={() => {
        if (typeof window !== "undefined" && window.history.length > 1) {
          router.history.back();
        } else {
          void router.navigate({ to: "/" });
        }
      }}
      className={`glass-chip inline-flex h-10 items-center gap-2 rounded-full px-4 text-sm font-medium text-muted-foreground transition hover:text-foreground ${className}`}
    >
      <ArrowLeft className="size-4" />
      {label}
    </button>
  );
}
```

### `src/components/isa/header.tsx`

133 righe

```tsx
import { Bell, LayoutGrid, List, Moon, Plus, Search, Sun } from "lucide-react";
import { useTheme } from "@/lib/theme";
import { IsaLogo } from "./logo";

export type CardView = "grid" | "list";

export function IsaHeader({
  query,
  onQueryChange,
  onNewSolution,
  view,
  onViewChange,
}: {
  query?: string | undefined;
  onQueryChange?: ((value: string) => void) | undefined;
  onNewSolution: () => void;
  view?: CardView | undefined;
  onViewChange?: ((view: CardView) => void) | undefined;
}) {
  const { theme, toggle } = useTheme();
  const isDark = theme === "dark";

  return (
    <header className="glass-panel flex flex-wrap items-center gap-2 rounded-3xl p-3">
      <span className="md:hidden">
        <IsaLogo showWordmark={false} />
      </span>

      {onQueryChange && (
        <label className="glass-chip flex h-10 min-w-0 flex-1 items-center gap-2 rounded-full px-3 lg:max-w-sm">
          <Search className="size-4 shrink-0 text-muted-foreground" />
          <input
            value={query ?? ""}
            onChange={(e) => onQueryChange(e.target.value)}
            placeholder="Cerca soluzioni…"
            aria-label="Cerca soluzioni"
            className="w-full bg-transparent text-sm outline-none placeholder:text-muted-foreground"
          />
        </label>
      )}

      <div className="ml-auto flex flex-wrap items-center gap-2">
        {view && onViewChange && (
          <div
            role="group"
            aria-label="Modalità di visualizzazione"
            className="glass-chip flex h-10 items-center gap-1 rounded-full p-1"
          >
            <ViewButton
              active={view === "grid"}
              label="Vista griglia"
              Icon={LayoutGrid}
              onClick={() => onViewChange("grid")}
            />
            <ViewButton
              active={view === "list"}
              label="Vista lista"
              Icon={List}
              onClick={() => onViewChange("list")}
            />
          </div>
        )}

        <button
          type="button"
          onClick={toggle}
          role="switch"
          aria-checked={isDark}
          aria-label={isDark ? "Passa alla modalità chiara" : "Passa alla modalità scura"}
          className="relative flex h-7 w-14 items-center rounded-full border border-white/20 bg-secondary/60 shadow-inner transition-colors duration-300 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring"
        >
          <span
            className="pointer-events-none absolute left-1 top-1 flex size-5 items-center justify-center rounded-full bg-white text-foreground shadow-sm transition-all duration-300 ease-[cubic-bezier(0.34,1.56,0.64,1)]"
            style={{ transform: isDark ? "translateX(28px)" : "translateX(0)" }}
          >
            {isDark ? <Moon className="size-3" /> : <Sun className="size-3" />}
          </span>
          <span className="flex w-full justify-between px-2">
            <Sun className="size-3 text-muted-foreground/60" />
            <Moon className="size-3 text-muted-foreground/60" />
          </span>
        </button>

        <button
          type="button"
          aria-label="Notifiche"
          className="glass-chip relative flex size-10 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <Bell className="size-[18px]" />
          <span className="absolute right-2 top-2 size-2 rounded-full bg-brand-glow" />
        </button>

        <button
          type="button"
          onClick={onNewSolution}
          className="gradient-brand flex h-10 items-center gap-2 rounded-full px-4 text-sm font-semibold text-brand-foreground shadow-[var(--shadow-glass)] transition hover:brightness-110"
        >
          <Plus className="size-4" />
          New solution
        </button>
      </div>
    </header>
  );
}

function ViewButton({
  active,
  label,
  Icon,
  onClick,
}: {
  active: boolean;
  label: string;
  Icon: typeof LayoutGrid;
  onClick: () => void;
}) {
  return (
    <button
      type="button"
      onClick={onClick}
      aria-label={label}
      aria-pressed={active}
      className={`flex size-8 items-center justify-center rounded-full transition ${
        active
          ? "gradient-brand text-brand-foreground"
          : "text-muted-foreground hover:text-foreground"
      }`}
    >
      <Icon className="size-4" />
    </button>
  );
}
```

### `src/components/isa/logo.tsx`

58 righe

```tsx
export function IsaLogo({
  className,
  showWordmark = true,
}: {
  className?: string;
  showWordmark?: boolean;
}) {
  return (
    <span className={`flex items-center gap-2.5 ${className ?? ""}`}>
      <svg
        width="36"
        height="36"
        viewBox="0 0 36 36"
        role="img"
        aria-label="isa"
        className="shrink-0 drop-shadow-sm"
      >
        <defs>
          <linearGradient id="isa-grad" x1="0" y1="0" x2="1" y2="1">
            <stop offset="0%" stopColor="var(--brand)" />
            <stop offset="100%" stopColor="var(--brand-glow)" />
          </linearGradient>
        </defs>
        <rect x="1.5" y="1.5" width="33" height="33" rx="10" fill="url(#isa-grad)" opacity="0.95" />
        <rect
          x="1.5"
          y="1.5"
          width="33"
          height="33"
          rx="10"
          fill="none"
          stroke="var(--glass-border)"
          strokeWidth="1.5"
        />
        <circle cx="9.5" cy="10" r="1.9" fill="var(--brand-foreground)" />
        <rect x="8" y="14" width="3" height="13" rx="1.5" fill="var(--brand-foreground)" />
        <path
          d="M15 25.5c1.6 1.2 5.6 1.5 5.6-1.3 0-2.7-5.3-2.3-5.3-5.2 0-2.6 3.7-2.7 5.3-1.7"
          fill="none"
          stroke="var(--brand-foreground)"
          strokeWidth="2.6"
          strokeLinecap="round"
        />
        <path
          d="M23.6 20.6c1.2-2.4 5.2-2.4 5.2.7v5.8m0-3.2c-3.7-.9-5.9.4-5.4 2.1.4 1.6 3.6 1.7 5.4-.3"
          fill="none"
          stroke="var(--brand-foreground)"
          strokeWidth="2.4"
          strokeLinecap="round"
        />
      </svg>
      {showWordmark && (
        <span className="text-2xl font-semibold tracking-tight text-gradient-brand">isa</span>
      )}
    </span>
  );
}
```

### `src/components/isa/mini-chart.tsx`

77 righe

```tsx
import type { ChartKind } from "@/lib/solutions-store";

export function MiniChart({
  data,
  kind,
  className,
}: {
  data: number[];
  kind: ChartKind;
  className?: string;
}) {
  const max = Math.max(...data, 1);
  const w = 100;
  const h = 40;

  if (kind === "bar") {
    const gap = 2.4;
    const bw = (w - gap * (data.length - 1)) / data.length;
    return (
      <svg viewBox={`0 0 ${w} ${h}`} preserveAspectRatio="none" className={className} aria-hidden>
        <defs>
          <linearGradient id="isa-bar" x1="0" y1="0" x2="0" y2="1">
            <stop offset="0%" stopColor="var(--brand-glow)" />
            <stop offset="100%" stopColor="var(--brand)" />
          </linearGradient>
        </defs>
        {data.map((v, i) => {
          const bh = Math.max(2, (v / max) * (h - 3));
          return (
            <rect
              key={i}
              x={i * (bw + gap)}
              y={h - bh}
              width={bw}
              height={bh}
              rx={1.6}
              fill="url(#isa-bar)"
              opacity={0.55 + (i / data.length) * 0.45}
            />
          );
        })}
      </svg>
    );
  }

  const points = data.map((v, i) => {
    const x = (i / (data.length - 1)) * w;
    const y = h - (v / max) * (h - 4) - 2;
    return `${x},${y}`;
  });

  return (
    <svg viewBox={`0 0 ${w} ${h}`} preserveAspectRatio="none" className={className} aria-hidden>
      <defs>
        <linearGradient id="isa-line" x1="0" y1="0" x2="1" y2="0">
          <stop offset="0%" stopColor="var(--brand)" />
          <stop offset="100%" stopColor="var(--brand-glow)" />
        </linearGradient>
        <linearGradient id="isa-fill" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%" stopColor="var(--brand-glow)" stopOpacity="0.45" />
          <stop offset="100%" stopColor="var(--brand)" stopOpacity="0" />
        </linearGradient>
      </defs>
      <polygon points={`0,${h} ${points.join(" ")} ${w},${h}`} fill="url(#isa-fill)" />
      <polyline
        points={points.join(" ")}
        fill="none"
        stroke="url(#isa-line)"
        strokeWidth="2"
        strokeLinecap="round"
        strokeLinejoin="round"
        vectorEffect="non-scaling-stroke"
      />
    </svg>
  );
}
```

### `src/components/isa/module-picker-modal.tsx`

74 righe

```tsx
import { Link } from "@tanstack/react-router";
import { ArrowUpRight, X } from "lucide-react";
import { MODULES } from "@/lib/modules";
import type { Solution } from "@/lib/solutions-store";

export function ModulePickerModal({
  solution,
  onClose,
}: {
  solution: Solution | null;
  onClose: () => void;
}) {
  if (!solution) return null;

  return (
    <div
      className="fixed inset-0 z-50 flex items-center justify-center p-4"
      role="dialog"
      aria-modal="true"
      aria-label={`Accedi a ${solution.name}`}
    >
      <button
        type="button"
        aria-label="Chiudi"
        onClick={onClose}
        className="absolute inset-0 bg-background/60 backdrop-blur-sm"
      />
      <div className="glass-panel relative w-full max-w-2xl rounded-3xl p-6">
        <button
          type="button"
          onClick={onClose}
          aria-label="Chiudi"
          className="glass-chip absolute right-4 top-4 flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <X className="size-4" />
        </button>

        <h2 className="text-lg font-semibold">Accedi a {solution.name}</h2>
        <p className="mt-1 text-sm text-muted-foreground">Seleziona il modulo su cui lavorare</p>

        <div className="mt-5 grid gap-3 sm:grid-cols-3">
          {MODULES.filter((m) => solution.modules[m.key]).map(
            ({ key, label, description, Icon, to }) => (
              <Link
                key={key}
                to={to}
                params={{ solutionId: solution.id }}
                onClick={onClose}
                className="glass-chip group flex flex-col gap-2 rounded-2xl p-4 text-left transition hover:-translate-y-0.5 hover:ring-2 hover:ring-ring"
              >
                <span
                  className={`badge-type-${key} flex size-10 items-center justify-center rounded-2xl`}
                >
                  <Icon className="size-5" />
                </span>
                <span className="flex items-center gap-1 text-sm font-semibold">
                  {label}
                  <ArrowUpRight className="size-3.5 opacity-0 transition group-hover:opacity-100" />
                </span>
                <span className="text-[11px] leading-snug text-muted-foreground">
                  {description}
                </span>
                <span className="mt-1 text-[10px] font-semibold uppercase tracking-wide text-muted-foreground">
                  {solution.modules[key] === "live" ? "Live" : "Draft"}
                </span>
              </Link>
            ),
          )}
        </div>
      </div>
    </div>
  );
}
```

### `src/components/isa/new-solution-modal.tsx`

206 righe

```tsx
import { Check, X } from "lucide-react";
import { useState } from "react";
import { MODULES } from "@/lib/modules";
import type { ModuleKey } from "@/lib/solutions-store";

const NUMBERS = Array.from({ length: 10 }, (_, i) => i);

export function NewSolutionModal({
  open,
  onClose,
  onCreate,
}: {
  open: boolean;
  onClose: () => void;
  onCreate: (input: {
    name: string;
    description: string;
    version: string;
    modules: ModuleKey[];
  }) => void;
}) {
  const [name, setName] = useState("");
  const [major, setMajor] = useState(1);
  const [minor, setMinor] = useState(0);
  const [patch, setPatch] = useState(0);
  const [description, setDescription] = useState("");
  const [modules, setModules] = useState<ModuleKey[]>([]);
  const [error, setError] = useState("");

  if (!open) return null;

  const toggleModule = (key: ModuleKey) => {
    setError("");
    setModules((prev) => (prev.includes(key) ? prev.filter((k) => k !== key) : [...prev, key]));
  };

  const submit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!name.trim()) return;
    if (modules.length === 0) {
      setError("Seleziona almeno un modulo.");
      return;
    }
    onCreate({
      name: name.trim(),
      description: description.trim(),
      version: `v${major}.${minor}.${patch}`,
      modules,
    });
    setName("");
    setMajor(1);
    setMinor(0);
    setPatch(0);
    setDescription("");
    setModules([]);
    setError("");
  };

  return (
    <div
      className="fixed inset-0 z-50 flex items-center justify-center p-4"
      role="dialog"
      aria-modal="true"
      aria-label="Crea nuova soluzione"
    >
      <button
        type="button"
        aria-label="Chiudi"
        onClick={onClose}
        className="absolute inset-0 bg-background/50 backdrop-blur-md"
      />
      <form onSubmit={submit} className="glass-panel relative w-full max-w-lg rounded-3xl p-6">
        <button
          type="button"
          onClick={onClose}
          aria-label="Chiudi"
          className="glass-chip absolute right-4 top-4 flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <X className="size-4" />
        </button>

        <h2 className="text-lg font-semibold">Nuova soluzione</h2>
        <p className="mt-1 text-sm text-muted-foreground">
          Scegli nome, versione e i moduli da attivare per questa soluzione.
        </p>

        <div className="mt-5 space-y-4">
          <div>
            <label htmlFor="sol-name" className="text-sm font-medium">
              Nome progetto <span className="text-destructive">*</span>
            </label>
            <input
              id="sol-name"
              value={name}
              onChange={(e) => setName(e.target.value)}
              placeholder="Es. Margine per prodotto"
              required
              className="glass-chip mt-1.5 h-11 w-full rounded-2xl px-3.5 text-sm outline-none focus:ring-2 focus:ring-ring"
            />
          </div>

          <div>
            <span className="text-sm font-medium">Versione</span>
            <div className="mt-1.5 flex items-center gap-2">
              <span className="text-sm font-semibold text-muted-foreground">v</span>
              <SemverSelect label="Major" value={major} onChange={setMajor} />
              <span className="text-muted-foreground">.</span>
              <SemverSelect label="Minor" value={minor} onChange={setMinor} />
              <span className="text-muted-foreground">.</span>
              <SemverSelect label="Patch" value={patch} onChange={setPatch} />
            </div>
          </div>

          <div>
            <span className="text-sm font-medium">
              Moduli <span className="text-destructive">*</span>
            </span>
            <div className="mt-1.5 grid gap-2 sm:grid-cols-3">
              {MODULES.map(({ key, label, description: desc, Icon }) => {
                const active = modules.includes(key);
                return (
                  <button
                    key={key}
                    type="button"
                    role="checkbox"
                    aria-checked={active}
                    onClick={() => toggleModule(key)}
                    className={`glass-chip relative flex flex-col gap-1.5 rounded-2xl p-3 text-left transition ${
                      active ? "ring-2 ring-ring" : "hover:-translate-y-0.5"
                    }`}
                  >
                    <span
                      className={`badge-type-${key} flex size-8 items-center justify-center rounded-xl`}
                    >
                      <Icon className="size-4" />
                    </span>
                    <span className="text-xs font-semibold">{label}</span>
                    <span className="text-[10px] leading-snug text-muted-foreground">{desc}</span>
                    {active && <Check className="absolute right-2 top-2 size-3.5 text-brand" />}
                  </button>
                );
              })}
            </div>
            {error && <p className="mt-2 text-xs text-destructive">{error}</p>}
          </div>

          <div>
            <label htmlFor="sol-desc" className="text-sm font-medium">
              Descrizione
            </label>
            <textarea
              id="sol-desc"
              value={description}
              onChange={(e) => setDescription(e.target.value)}
              rows={3}
              placeholder="Cosa calcola questa soluzione?"
              className="glass-chip mt-1.5 w-full resize-none rounded-2xl p-3.5 text-sm outline-none focus:ring-2 focus:ring-ring"
            />
          </div>
        </div>

        <div className="mt-6 flex justify-end gap-2">
          <button
            type="button"
            onClick={onClose}
            className="glass-chip h-10 rounded-full px-4 text-sm font-medium"
          >
            Annulla
          </button>
          <button
            type="submit"
            className="gradient-brand h-10 rounded-full px-5 text-sm font-semibold text-brand-foreground transition hover:brightness-110"
          >
            Crea soluzione
          </button>
        </div>
      </form>
    </div>
  );
}

function SemverSelect({
  label,
  value,
  onChange,
}: {
  label: string;
  value: number;
  onChange: (value: number) => void;
}) {
  return (
    <select
      aria-label={label}
      value={value}
      onChange={(e) => onChange(Number(e.target.value))}
      className="glass-chip h-11 rounded-2xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
    >
      {NUMBERS.map((n) => (
        <option key={n} value={n}>
          {n}
        </option>
      ))}
    </select>
  );
}
```

### `src/components/isa/share-modal.tsx`

123 righe

```tsx
import { Trash2, UserPlus, X } from "lucide-react";
import { useState } from "react";
import { type SharePermission, type Solution, useSolutions } from "@/lib/solutions-store";

export function ShareModal({
  solution,
  onClose,
}: {
  solution: Solution | null;
  onClose: () => void;
}) {
  const { addShare, updateShare, removeShare } = useSolutions();
  const [email, setEmail] = useState("");
  const [permission, setPermission] = useState<SharePermission>("view");

  if (!solution) return null;

  const submit = (e: React.FormEvent) => {
    e.preventDefault();
    if (!email.trim()) return;
    addShare(solution.id, email, permission);
    setEmail("");
  };

  return (
    <div
      className="fixed inset-0 z-50 flex items-center justify-center p-4"
      role="dialog"
      aria-modal="true"
      aria-label={`Condividi ${solution.name}`}
    >
      <button
        type="button"
        aria-label="Chiudi"
        onClick={onClose}
        className="absolute inset-0 bg-background/60 backdrop-blur-sm"
      />
      <div className="glass-panel relative w-full max-w-lg rounded-3xl p-6">
        <button
          type="button"
          onClick={onClose}
          aria-label="Chiudi"
          className="glass-chip absolute right-4 top-4 flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <X className="size-4" />
        </button>

        <h2 className="text-lg font-semibold">Condividi {solution.name}</h2>
        <p className="mt-1 text-sm text-muted-foreground">
          Invita persone o team e scegli i permessi di accesso.
        </p>

        <form onSubmit={submit} className="mt-5 flex flex-wrap gap-2">
          <input
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            placeholder="nome@azienda.com"
            aria-label="Email da invitare"
            className="glass-chip h-11 min-w-0 flex-1 rounded-2xl px-3.5 text-sm outline-none focus:ring-2 focus:ring-ring"
          />
          <select
            value={permission}
            onChange={(e) => setPermission(e.target.value as SharePermission)}
            aria-label="Permesso"
            className="glass-chip h-11 rounded-2xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
          >
            <option value="view">Visualizzazione</option>
            <option value="edit">Modifica</option>
          </select>
          <button
            type="submit"
            className="gradient-brand flex h-11 items-center gap-2 rounded-2xl px-4 text-sm font-semibold text-brand-foreground transition hover:brightness-110"
          >
            <UserPlus className="size-4" />
            Invita
          </button>
        </form>

        <div className="mt-5 space-y-2">
          <span className="text-xs font-semibold uppercase tracking-wide text-muted-foreground">
            Accesso ({solution.shares.length + 1})
          </span>
          <div className="glass-chip flex items-center justify-between rounded-2xl px-3.5 py-2.5 text-sm">
            <span className="truncate">{solution.owner} (proprietario)</span>
            <span className="text-xs text-muted-foreground">Tutti i permessi</span>
          </div>
          {solution.shares.map((share) => (
            <div
              key={share.id}
              className="glass-chip flex items-center gap-2 rounded-2xl px-3.5 py-2.5 text-sm"
            >
              <span className="min-w-0 flex-1 truncate">{share.email}</span>
              <select
                value={share.permission}
                onChange={(e) =>
                  updateShare(solution.id, share.id, e.target.value as SharePermission)
                }
                aria-label={`Permesso di ${share.email}`}
                className="glass-chip h-9 rounded-xl px-2 text-xs outline-none focus:ring-2 focus:ring-ring"
              >
                <option value="view">Visualizzazione</option>
                <option value="edit">Modifica</option>
              </select>
              <button
                type="button"
                onClick={() => removeShare(solution.id, share.id)}
                aria-label={`Rimuovi ${share.email}`}
                className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-destructive"
              >
                <Trash2 className="size-4" />
              </button>
            </div>
          ))}
          {solution.shares.length === 0 && (
            <p className="text-xs text-muted-foreground">Nessuna condivisione attiva.</p>
          )}
        </div>
      </div>
    </div>
  );
}
```

### `src/components/isa/sidebar.tsx`

114 righe

```tsx
import { Link } from "@tanstack/react-router";
import {
  Activity,
  ChevronLeft,
  Cog,
  Grid2x2,
  LayoutGrid,
  Star,
  Trash2,
  Users,
  UsersRound,
  Share2,
} from "lucide-react";
import { useState } from "react";
import { IsaLogo } from "./logo";

const primary = [
  { label: "Solutions", to: "/", icon: LayoutGrid },
  { label: "Shared with me", to: "/shared", icon: Share2 },
  { label: "Favorites", to: "/favorites", icon: Star },
  { label: "Templates", to: "/templates", icon: Grid2x2 },
  { label: "Activity", to: "/activity", icon: Activity },
  { label: "Trash", to: "/trash", icon: Trash2 },
] as const;

const secondary = [
  { label: "Teams", to: "/teams", icon: UsersRound },
  { label: "Users", to: "/users", icon: Users },
  { label: "Settings", to: "/settings", icon: Cog },
] as const;

export function IsaSidebar() {
  const [collapsed, setCollapsed] = useState(false);

  return (
    <aside
      className={`glass-panel sticky top-4 hidden h-[calc(100vh-2rem)] shrink-0 flex-col rounded-3xl p-3 transition-all duration-300 md:flex ${
        collapsed ? "w-[76px]" : "w-64"
      }`}
    >
      <div className="flex items-center justify-between px-1 py-2">
        <IsaLogo showWordmark={!collapsed} />
        <button
          type="button"
          onClick={() => setCollapsed((c) => !c)}
          aria-label={collapsed ? "Espandi menu" : "Comprimi menu"}
          className="glass-chip flex size-7 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <ChevronLeft className={`size-4 transition-transform ${collapsed ? "rotate-180" : ""}`} />
        </button>
      </div>

      <nav className="mt-4 flex flex-1 flex-col gap-6 overflow-y-auto scroll-slim">
        <NavGroup title="Workspace" items={primary} collapsed={collapsed} />
        <NavGroup title="Organizzazione" items={secondary} collapsed={collapsed} />
      </nav>

      <Link
        to="/settings"
        className="glass-chip mt-3 flex items-center gap-3 rounded-2xl p-2 transition hover:brightness-105"
      >
        <span className="gradient-brand flex size-9 shrink-0 items-center justify-center rounded-full text-sm font-semibold text-brand-foreground">
          MF
        </span>
        {!collapsed && (
          <span className="min-w-0">
            <span className="block truncate text-sm font-medium">Michele Franco</span>
            <span className="block truncate text-xs text-muted-foreground">Workspace admin</span>
          </span>
        )}
      </Link>
    </aside>
  );
}

function NavGroup({
  title,
  items,
  collapsed,
}: {
  title: string;
  items: readonly { label: string; to: string; icon: React.ElementType }[];
  collapsed: boolean;
}) {
  return (
    <div>
      {!collapsed && (
        <p className="px-3 pb-2 text-[11px] font-semibold uppercase tracking-widest text-muted-foreground">
          {title}
        </p>
      )}
      <ul className="flex flex-col gap-1">
        {items.map((item) => (
          <li key={item.label}>
            <Link
              to={item.to}
              activeOptions={{ exact: item.to === "/" }}
              title={item.label}
              className="flex items-center gap-3 rounded-2xl px-3 py-2.5 text-sm font-medium text-sidebar-foreground/85 transition hover:bg-sidebar-accent hover:text-foreground"
              activeProps={{
                className:
                  "bg-sidebar-accent text-foreground shadow-[inset_0_1px_0_var(--glass-border)]",
              }}
            >
              <item.icon className="size-[18px] shrink-0" />
              {!collapsed && <span className="truncate">{item.label}</span>}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### `src/components/isa/solution-card.tsx`

215 righe

```tsx
import {
  CalendarClock,
  Check,
  Copy,
  Pencil,
  Plus,
  Share2,
  Tag,
  ToggleLeft,
  Trash2,
} from "lucide-react";
import { useState } from "react";
import { IsaMenu, IsaMenuItem } from "@/components/isa/ui/isa-menu";
import { MODULES } from "@/lib/modules";
import { type Solution, useSolutions } from "@/lib/solutions-store";

export function SolutionCard({
  solution,
  onDelete,
  onOpen,
  onShare,
}: {
  solution: Solution;
  onDelete: (id: string) => void;
  onOpen: (solution: Solution) => void;
  onShare: (solution: Solution) => void;
}) {
  const { updateSolution, duplicateSolution } = useSolutions();
  const [editing, setEditing] = useState<"name" | "version" | null>(null);
  const [draft, setDraft] = useState("");
  const live = solution.status === "live";

  const startEdit = (field: "name" | "version", close: () => void) => {
    setDraft(field === "name" ? solution.name : solution.version);
    setEditing(field);
    close();
  };

  const commitEdit = (e: React.FormEvent) => {
    e.preventDefault();
    const value = draft.trim();
    if (value) {
      updateSolution(solution.id, editing === "name" ? { name: value } : { version: value });
    }
    setEditing(null);
  };

  const stop = (e: React.MouseEvent) => e.stopPropagation();

  return (
    <div className="glass-panel group relative flex flex-col rounded-3xl p-5 transition duration-300 hover:-translate-y-1">
      <button
        type="button"
        onClick={() => onOpen(solution)}
        aria-label={`Apri ${solution.name}`}
        className="absolute inset-0 z-0 rounded-3xl focus:outline-none focus-visible:ring-2 focus-visible:ring-ring"
      />

      <div className="pointer-events-none relative z-20 flex items-start justify-between gap-3">
        <div className="min-w-0">
          {editing ? (
            <form onSubmit={commitEdit} onClick={stop} className="pointer-events-auto flex gap-2">
              <input
                autoFocus
                value={draft}
                onChange={(e) => setDraft(e.target.value)}
                onBlur={() => setEditing(null)}
                aria-label={editing === "name" ? "Nuovo nome" : "Nuova versione"}
                className="glass-chip h-9 min-w-0 flex-1 rounded-xl px-3 text-sm outline-none focus:ring-2 focus:ring-ring"
              />
              <button
                type="submit"
                aria-label="Salva"
                className="glass-chip flex size-9 items-center justify-center rounded-xl text-brand"
              >
                <Check className="size-4" />
              </button>
            </form>
          ) : (
            <span className="block truncate text-base font-semibold group-hover:text-brand">
              {solution.name}
            </span>
          )}
          <p className="mt-1 line-clamp-2 text-sm text-muted-foreground">{solution.description}</p>
          <span className="glass-chip mt-2 inline-block rounded-full px-2.5 py-1 text-[11px] text-muted-foreground">
            {solution.version}
          </span>
        </div>

        <div className="pointer-events-auto flex shrink-0 items-center gap-1.5" onClick={stop}>
          <button
            type="button"
            onClick={() => updateSolution(solution.id, { status: live ? "draft" : "live" })}
            role="switch"
            aria-checked={live}
            aria-label={`Stato: ${live ? "Live" : "Draft"}`}
            className="glass-chip rounded-full px-2.5 py-1 text-[11px] font-semibold uppercase tracking-wide transition"
            style={{ color: live ? "var(--success)" : "var(--warning)" }}
          >
            {live ? "Live" : "Draft"}
          </button>

          <IsaMenu label={`Gestisci ${solution.name}`}>
            {(close) => (
              <>
                <IsaMenuItem
                  Icon={ToggleLeft}
                  label={live ? "Imposta come Draft" : "Imposta come Live"}
                  onClick={() => {
                    updateSolution(solution.id, { status: live ? "draft" : "live" });
                    close();
                  }}
                />
                <IsaMenuItem
                  Icon={Pencil}
                  label="Rinomina"
                  onClick={() => startEdit("name", close)}
                />
                <IsaMenuItem
                  Icon={Tag}
                  label="Modifica versione"
                  onClick={() => startEdit("version", close)}
                />
                <IsaMenuItem
                  Icon={Copy}
                  label="Duplica"
                  onClick={() => {
                    duplicateSolution(solution.id);
                    close();
                  }}
                />
                <IsaMenuItem
                  Icon={Share2}
                  label="Condividi"
                  onClick={() => {
                    onShare(solution);
                    close();
                  }}
                />
                <IsaMenuItem
                  Icon={Trash2}
                  label="Elimina"
                  danger
                  onClick={() => {
                    close();
                    onDelete(solution.id);
                  }}
                />
              </>
            )}
          </IsaMenu>
        </div>
      </div>

      <div className="pointer-events-none relative z-10 mt-3 flex flex-wrap gap-1.5">
        {MODULES.filter((m) => solution.modules[m.key]).map(({ key, label, Icon }) => (
          <span
            key={key}
            className={`badge-type-${key} flex items-center gap-1.5 rounded-full px-2.5 py-1 text-[11px] font-semibold`}
          >
            <Icon className="size-3.5" />
            {label}
            <span className="opacity-70">
              {solution.modules[key] === "live" ? "· Live" : "· Draft"}
            </span>
          </span>
        ))}
      </div>

      <div className="relative z-10 mt-4 flex items-center justify-between">
        <span className="pointer-events-none flex items-center gap-1.5 text-xs text-muted-foreground">
          <CalendarClock className="size-3.5" />
          Aggiornata il {solution.updatedAt}
        </span>
        <div className="pointer-events-auto flex items-center gap-1.5" onClick={stop}>
          <button
            type="button"
            onClick={() => onShare(solution)}
            aria-label={`Condividi ${solution.name}`}
            className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
          >
            <Share2 className="size-4" />
          </button>
          <button
            type="button"
            onClick={() => onDelete(solution.id)}
            aria-label={`Elimina ${solution.name}`}
            className="glass-chip flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-destructive"
          >
            <Trash2 className="size-4" />
          </button>
        </div>
      </div>
    </div>
  );
}

export function NewSolutionCard({ onClick }: { onClick: () => void }) {
  return (
    <button
      type="button"
      onClick={onClick}
      className="glass-soft flex min-h-[260px] flex-col items-center justify-center gap-3 rounded-3xl border-dashed p-5 text-muted-foreground transition hover:-translate-y-1 hover:text-foreground"
    >
      <span className="gradient-brand flex size-12 items-center justify-center rounded-2xl text-brand-foreground">
        <Plus className="size-6" />
      </span>
      <span className="text-sm font-semibold">Crea nuova soluzione</span>
      <span className="max-w-[200px] text-center text-xs">
        ETL, Model &amp; Algorithm e Dashboard inclusi
      </span>
    </button>
  );
}
```

### `src/components/isa/solution-row.tsx`

135 righe

```tsx
import { CalendarClock, Copy, Pencil, Share2, Tag, ToggleLeft, Trash2 } from "lucide-react";
import { IsaMenu, IsaMenuItem } from "@/components/isa/ui/isa-menu";
import { MODULES } from "@/lib/modules";
import { type Solution, useSolutions } from "@/lib/solutions-store";

export function SolutionRow({
  solution,
  onOpen,
  onDelete,
  onShare,
}: {
  solution: Solution;
  onOpen: (solution: Solution) => void;
  onDelete: (id: string) => void;
  onShare: (solution: Solution) => void;
}) {
  const { updateSolution, duplicateSolution } = useSolutions();
  const live = solution.status === "live";
  const stop = (e: React.MouseEvent) => e.stopPropagation();

  const promptEdit = (field: "name" | "version", close: () => void) => {
    const current = field === "name" ? solution.name : solution.version;
    const value = window.prompt(
      field === "name" ? "Nuovo nome della soluzione" : "Nuova versione",
      current,
    );
    close();
    if (value && value.trim()) {
      updateSolution(
        solution.id,
        field === "name" ? { name: value.trim() } : { version: value.trim() },
      );
    }
  };

  return (
    <div className="glass-panel relative flex flex-wrap items-center gap-3 rounded-2xl p-3">
      <button
        type="button"
        onClick={() => onOpen(solution)}
        aria-label={`Apri ${solution.name}`}
        className="absolute inset-0 rounded-2xl focus:outline-none focus-visible:ring-2 focus-visible:ring-ring"
      />

      <div className="pointer-events-none relative min-w-0 flex-1">
        <span className="block truncate text-sm font-semibold">{solution.name}</span>
        <span className="flex items-center gap-1.5 text-[11px] text-muted-foreground">
          <CalendarClock className="size-3" />
          {solution.updatedAt}
        </span>
      </div>

      <div className="pointer-events-none relative flex flex-wrap items-center gap-1.5">
        {MODULES.filter((m) => solution.modules[m.key]).map(({ key, label, Icon }) => (
          <span
            key={key}
            className={`badge-type-${key} flex items-center gap-1 rounded-full px-2 py-1 text-[10px] font-semibold`}
          >
            <Icon className="size-3" />
            {label}
          </span>
        ))}
      </div>

      <span className="glass-chip pointer-events-none relative rounded-full px-2.5 py-1 text-[11px] text-muted-foreground">
        {solution.version}
      </span>

      <div className="relative flex items-center gap-1.5" onClick={stop}>
        <button
          type="button"
          onClick={() => updateSolution(solution.id, { status: live ? "draft" : "live" })}
          role="switch"
          aria-checked={live}
          aria-label={`Stato di ${solution.name}: ${live ? "Live" : "Draft"}`}
          className="glass-chip rounded-full px-2.5 py-1 text-[11px] font-semibold uppercase tracking-wide"
          style={{ color: live ? "var(--success)" : "var(--warning)" }}
        >
          {live ? "Live" : "Draft"}
        </button>

        <IsaMenu label={`Gestisci ${solution.name}`}>
          {(close) => (
            <>
              <IsaMenuItem
                Icon={ToggleLeft}
                label={live ? "Imposta come Draft" : "Imposta come Live"}
                onClick={() => {
                  updateSolution(solution.id, { status: live ? "draft" : "live" });
                  close();
                }}
              />
              <IsaMenuItem
                Icon={Pencil}
                label="Rinomina"
                onClick={() => promptEdit("name", close)}
              />
              <IsaMenuItem
                Icon={Tag}
                label="Modifica versione"
                onClick={() => promptEdit("version", close)}
              />
              <IsaMenuItem
                Icon={Copy}
                label="Duplica"
                onClick={() => {
                  duplicateSolution(solution.id);
                  close();
                }}
              />
              <IsaMenuItem
                Icon={Share2}
                label="Condividi"
                onClick={() => {
                  onShare(solution);
                  close();
                }}
              />
              <IsaMenuItem
                Icon={Trash2}
                label="Elimina"
                danger
                onClick={() => {
                  close();
                  onDelete(solution.id);
                }}
              />
            </>
          )}
        </IsaMenu>
      </div>
    </div>
  );
}
```

### `src/components/isa/ui/isa-menu.tsx`

420 righe

```tsx
import type { Pencil } from "lucide-react";
import { Check, MoreHorizontal } from "lucide-react";
import { createPortal } from "react-dom";
import { useCallback, useEffect, useLayoutEffect, useRef, useState } from "react";
import {
  PRESS_SCALE,
  PRESS_TRANSITION,
  SPRING_CLOSE_DURATION_MS,
  SPRING_CLOSE_TRANSITION,
  SPRING_ORIGIN_SCALE,
  SPRING_OPEN_TRANSITION,
} from "@/lib/etl-motion";

type MenuSide = "top" | "bottom" | "left" | "right";

type MenuPlacement = "auto" | MenuSide;

type MenuPosition = {
  top: number;
  left: number;
};

const MENU_WIDTH = 208;
const MENU_GAP = 8;
const MENU_MARGIN = 8;

export function IsaMenu({
  label,
  children,
  align = "right",
  Icon = MoreHorizontal,
  triggerClassName = "",
  boundaryRef,
  placement = "bottom",
  variant = "chip",
  width = MENU_WIDTH,
  onOpenChange,
}: {
  label: string;
  children: (close: () => void) => React.ReactNode;
  align?: "left" | "right";
  Icon?: typeof Pencil;
  triggerClassName?: string;
  boundaryRef?: React.RefObject<HTMLElement | null>;
  placement?: MenuPlacement;
  variant?: "chip" | "bare";
  /** Larghezza del popover in px (default MENU_WIDTH) — es. per i pannelli impostazioni della fase 4, più larghi di un menu azioni. */
  width?: number | undefined;
  /**
   * Notifica il chiamante quando il menu si apre/chiude — usato da chi
   * ospita il trigger (es. una card nodo) per applicare il proprio
   * feedback "pressed" all'intero contenitore, non solo al bottone
   * icona (punto 2.1 del redesign iOS).
   */
  onOpenChange?: (open: boolean) => void;
}) {
  const [open, setOpen] = useState(false);

  /*
   * `rendered` tiene il popover nel DOM anche durante la chiusura, per
   * poter animare il rientro verso il trigger invece di sparire di
   * scatto; `visible` guida la transizione scale/opacity vera e propria
   * e viene alzata un frame dopo il mount (serve un primo commit con lo
   * stato "piccolo/trasparente" prima di animare verso quello finale).
   */
  const [rendered, setRendered] = useState(false);

  const [visible, setVisible] = useState(false);

  const [side, setSide] = useState<MenuSide>("bottom");

  const [position, setPosition] = useState<MenuPosition | null>(null);

  const rootRef = useRef<HTMLDivElement>(null);

  const triggerRef = useRef<HTMLButtonElement>(null);

  const menuRef = useRef<HTMLDivElement>(null);

  const closeTimeoutRef = useRef<number | null>(null);

  const constrained = Boolean(boundaryRef && placement === "auto");

  const clearCloseTimeout = useCallback(() => {
    if (closeTimeoutRef.current !== null) {
      window.clearTimeout(closeTimeoutRef.current);
      closeTimeoutRef.current = null;
    }
  }, []);

  const close = useCallback(() => {
    setOpen(false);
    onOpenChange?.(false);
  }, [onOpenChange]);

  /*
   * Sequenza di chiusura: appena `open` torna false il popover resta
   * montato (`rendered`) ma `visible` si abbassa, innescando la
   * transizione di rientro verso il trigger; solo al termine di quella
   * transizione lo smontiamo davvero e liberiamo `position`. Se il
   * trigger viene ripremuto durante il rientro, `clearCloseTimeout` (nel
   * toggle) annulla lo smontaggio pendente.
   */
  useEffect(() => {
    if (open || !rendered) {
      return;
    }

    setVisible(false);
    clearCloseTimeout();

    closeTimeoutRef.current = window.setTimeout(() => {
      setRendered(false);
      setPosition(null);
      closeTimeoutRef.current = null;
    }, SPRING_CLOSE_DURATION_MS);

    return clearCloseTimeout;
  }, [open, rendered, clearCloseTimeout]);

  const calculatePosition = useCallback(() => {
    if (!constrained) {
      return;
    }

    const trigger = triggerRef.current;

    const menu = menuRef.current;

    const boundary = boundaryRef?.current;

    if (!trigger || !menu || !boundary) {
      return;
    }

    const triggerRect = trigger.getBoundingClientRect();

    const menuRect = menu.getBoundingClientRect();

    const boundaryRect = boundary.getBoundingClientRect();

    const menuWidth = menuRect.width || width;

    const menuHeight = menuRect.height;

    const available: Record<MenuSide, number> = {
      top: triggerRect.top - boundaryRect.top - MENU_GAP - MENU_MARGIN,

      bottom: boundaryRect.bottom - triggerRect.bottom - MENU_GAP - MENU_MARGIN,

      left: triggerRect.left - boundaryRect.left - MENU_GAP - MENU_MARGIN,

      right: boundaryRect.right - triggerRect.right - MENU_GAP - MENU_MARGIN,
    };

    const fits: Record<MenuSide, boolean> = {
      top: available.top >= menuHeight,

      bottom: available.bottom >= menuHeight,

      left: available.left >= menuWidth,

      right: available.right >= menuWidth,
    };

    /*
     * Ordine preferenziale:
     * prima sotto, poi sopra, poi lato destro/sinistro.
     *
     * Se un lato non ha spazio sufficiente,
     * viene provato il successivo.
     */
    const order: MenuSide[] = ["bottom", "top", "right", "left"];

    let side: MenuSide | null = null;

    for (const candidate of order) {
      if (fits[candidate]) {
        side = candidate;
        break;
      }
    }

    /*
     * Se il Canvas è troppo piccolo per contenere
     * il menu interamente su un lato, scegliamo il lato
     * con più spazio e facciamo un clamp finale.
     */
    if (!side) {
      const fallback = (Object.keys(available) as MenuSide[]).sort(
        (a, b) => available[b] - available[a],
      );

      side = fallback[0] ?? "bottom";
    }

    let left = triggerRect.right - menuWidth;

    let top = triggerRect.bottom + MENU_GAP;

    if (side === "bottom") {
      top = triggerRect.bottom + MENU_GAP;

      left = align === "left" ? triggerRect.left : triggerRect.right - menuWidth;
    }

    if (side === "top") {
      top = triggerRect.top - menuHeight - MENU_GAP;

      left = align === "left" ? triggerRect.left : triggerRect.right - menuWidth;
    }

    if (side === "right") {
      left = triggerRect.right + MENU_GAP;

      top = triggerRect.top;
    }

    if (side === "left") {
      left = triggerRect.left - menuWidth - MENU_GAP;

      top = triggerRect.top;
    }

    const minLeft = boundaryRect.left + MENU_MARGIN;

    const maxLeft = Math.max(minLeft, boundaryRect.right - menuWidth - MENU_MARGIN);

    const minTop = boundaryRect.top + MENU_MARGIN;

    const maxTop = Math.max(minTop, boundaryRect.bottom - menuHeight - MENU_MARGIN);

    left = Math.min(Math.max(left, minLeft), maxLeft);

    top = Math.min(Math.max(top, minTop), maxTop);

    setPosition({
      left,
      top,
    });
  }, [align, boundaryRef, constrained, width]);

  useLayoutEffect(() => {
    if (!open || !constrained) {
      return;
    }

    calculatePosition();

    const frame = requestAnimationFrame(calculatePosition);

    return () => cancelAnimationFrame(frame);
  }, [open, constrained, calculatePosition]);

  useEffect(() => {
    if (!open) {
      return;
    }

    const handlePointerDown = (event: PointerEvent) => {
      const target = event.target as Node;

      const insideTrigger = rootRef.current?.contains(target);

      const insideMenu = menuRef.current?.contains(target);

      if (!insideTrigger && !insideMenu) {
        close();
      }
    };

    const handleKeyDown = (event: KeyboardEvent) => {
      if (event.key === "Escape") {
        close();
      }
    };

    document.addEventListener("pointerdown", handlePointerDown);

    document.addEventListener("keydown", handleKeyDown);

    return () => {
      document.removeEventListener("pointerdown", handlePointerDown);

      document.removeEventListener("keydown", handleKeyDown);
    };
  }, [open, close]);

  useEffect(() => {
    if (!open || !constrained) {
      return;
    }

    const reposition = () => {
      calculatePosition();
    };

    window.addEventListener("resize", reposition);

    window.addEventListener("scroll", reposition, true);

    return () => {
      window.removeEventListener("resize", reposition);

      window.removeEventListener("scroll", reposition, true);
    };
  }, [open, constrained, calculatePosition]);

  const menu = open ? (
    <div
      ref={menuRef}
      role="menu"
      onPointerDown={(event) => event.stopPropagation()}
      className={
        constrained
          ? "fixed z-[80] overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl"
          : `absolute ${
              align === "right" ? "right-0" : "left-0"
            } top-10 z-30 overflow-hidden rounded-2xl border border-border bg-background p-1.5 text-sm text-foreground shadow-xl`
      }
      style={
        constrained
          ? {
              width,
              left: position?.left ?? -10000,

              top: position?.top ?? -10000,

              visibility: position ? "visible" : "hidden",
            }
          : { width }
      }
    >
      {children(close)}
    </div>
  ) : null;

  return (
    <div ref={rootRef} className="relative">
      <button
        ref={triggerRef}
        type="button"
        onClick={() => {
          if (open) {
            close();
          } else {
            setOpen(true);
          }
        }}
        aria-label={label}
        aria-expanded={open}
        className={`flex size-8 items-center justify-center transition hover:text-foreground ${
          variant === "chip"
            ? "glass-chip rounded-full text-muted-foreground"
            : "text-muted-foreground"
        } ${triggerClassName}`}
      >
        <Icon className="size-4" />
      </button>

      {constrained && typeof document !== "undefined" ? createPortal(menu, document.body) : menu}
    </div>
  );
}

export function IsaMenuItem({
  Icon,
  label,
  onClick,
  danger,
}: {
  Icon: typeof Pencil;
  label: string;
  onClick: () => void;
  danger?: boolean;
}) {
  return (
    <button
      type="button"
      onClick={onClick}
      className={`flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left transition hover:bg-muted ${
        danger ? "text-destructive" : "text-foreground"
      }`}
    >
      <Icon className="size-4" />
      {label}
    </button>
  );
}

export function IsaMenuCheckItem({
  label,
  checked,
  onToggle,
}: {
  label: string;
  checked: boolean;
  onToggle: () => void;
}) {
  return (
    <button
      type="button"
      role="menuitemcheckbox"
      aria-checked={checked}
      onClick={onToggle}
      className="flex w-full items-center gap-2 rounded-xl px-3 py-2 text-left text-foreground transition hover:bg-muted"
    >
      <span
        className={`flex size-4 shrink-0 items-center justify-center rounded-[5px] border ${
          checked ? "gradient-brand border-transparent text-brand-foreground" : "border-border"
        }`}
      >
        {checked && <Check className="size-3" />}
      </span>

      <span className="min-w-0 flex-1 truncate text-[13px]">{label}</span>
    </button>
  );
}
```

### `src/components/isa/ui/isa-modal.tsx`

53 righe

```tsx
import { X } from "lucide-react";

/**
 * Shell modale canonica di isa (estratta da NewSolutionModal / ShareModal):
 * backdrop sfocato, pannello glass rounded-3xl, chiusura in alto a destra.
 */
export function IsaModal({
  open,
  onClose,
  title,
  subtitle,
  maxWidth = "max-w-lg",
  children,
}: {
  open: boolean;
  onClose: () => void;
  title: string;
  subtitle?: string;
  maxWidth?: string;
  children: React.ReactNode;
}) {
  if (!open) return null;

  return (
    <div
      className="fixed inset-0 z-50 flex items-center justify-center p-4"
      role="dialog"
      aria-modal="true"
      aria-label={title}
    >
      <button
        type="button"
        aria-label="Chiudi"
        onClick={onClose}
        className="absolute inset-0 bg-background/50 backdrop-blur-md"
      />
      <div className={`glass-panel relative w-full ${maxWidth} rounded-3xl p-6`}>
        <button
          type="button"
          onClick={onClose}
          aria-label="Chiudi"
          className="glass-chip absolute right-4 top-4 flex size-8 items-center justify-center rounded-full text-muted-foreground transition hover:text-foreground"
        >
          <X className="size-4" />
        </button>
        <h2 className="text-lg font-semibold">{title}</h2>
        {subtitle && <p className="mt-1 text-sm text-muted-foreground">{subtitle}</p>}
        <div className="mt-5">{children}</div>
      </div>
    </div>
  );
}
```

