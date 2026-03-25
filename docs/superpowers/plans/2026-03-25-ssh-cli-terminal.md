# SSH CLI + Terminal Tab — Plano de Implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adicionar uma aba "Terminal" ao site talesmiguel.dev e criar uma CLI acessível via SSH (estilo terminal.shop) usando Go + Bubble Tea + Wish, hospedada gratuitamente no fly.io.

**Architecture:** O projeto se divide em duas partes independentes: (1) modificações no site Astro (nova seção Terminal + hint na Hero) e (2) uma aplicação Go que serve uma TUI via SSH. A CLI replica o conteúdo do site (bio, skills, experience, education, contact) com uma interface de terminal interativa. O deploy da CLI vai para o fly.io (free tier), acessível via `ssh ssh.talesmiguel.dev`.

**Tech Stack:**
- Web: Astro 6, Tailwind v4, GSAP (já existente)
- CLI: Go, Charm Wish (SSH server), Bubble Tea (TUI framework), Lip Gloss (styling)
- Deploy: Vercel (site) + fly.io (CLI SSH)

---

## Ações manuais do usuário (Tales)

> Estas são ações que **só o Tales pode fazer** — cadastros, instalações de sistema, configurações de DNS. Estão marcadas com 🔧 ao longo do plano e listadas aqui de forma consolidada.

### 🔧 M1: Instalar Go

```bash
# Opção 1: via tarball oficial (recomendado)
wget https://go.dev/dl/go1.24.1.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.24.1.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.zshrc
source ~/.zshrc
go version  # deve mostrar go1.24.1

# Opção 2: via snap
sudo snap install go --classic
```

### 🔧 M2: Criar conta no fly.io e instalar flyctl

```bash
# 1. Criar conta em https://fly.io/app/sign-up (precisa de cartão, mas não cobra)

# 2. Instalar flyctl
curl -L https://fly.io/install.sh | sh
echo 'export FLYCTL_INSTALL="/home/tales/.fly"' >> ~/.zshrc
echo 'export PATH="$FLYCTL_INSTALL/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 3. Login
flyctl auth login
```

### 🔧 M3: Configurar DNS para ssh.talesmiguel.dev

Após o primeiro deploy no fly.io (Task 8), o fly.io vai te dar um IP. Tu precisas:

1. Ir no painel DNS do teu domínio (provavelmente onde compraste o domínio — Cloudflare, Namecheap, etc.)
2. Adicionar um registro **A** apontando `ssh.talesmiguel.dev` para o IP do fly.io
3. Ou usar o CNAME que o fly.io fornecer

```
Tipo: A
Nome: ssh
Valor: <IP do fly.io>
TTL: 300
```

> Nota: Se o domínio usa Cloudflare como DNS, desativa o proxy (nuvem laranja → cinza) para o subdomínio `ssh`, pois SSH precisa de conexão TCP direta.

### 🔧 M4: Criar repositório para a CLI (opcional)

Pode ser um novo repo (ex: `TalesMiguel/talesmiguel-ssh`) ou uma pasta no monorepo. Recomendo repo separado para deploy independente no fly.io.

```bash
# Se repo separado:
mkdir -p /home/tales/projects/talesmiguel-ssh
cd /home/tales/projects/talesmiguel-ssh
git init
gh repo create TalesMiguel/talesmiguel-ssh --private --source=. --remote=origin
```

---

## Mapa de Arquivos

### Parte 1: Site Astro (modificações)

| Ação | Arquivo | Responsabilidade |
|------|---------|-----------------|
| Criar | `src/components/Terminal.astro` | Seção Terminal com estética de terminal, mostra comando SSH |
| Modificar | `src/components/Hero.astro` | Adicionar hint "If you're a developer..." |
| Modificar | `src/pages/index.astro` | Importar e incluir componente Terminal como primeira seção |
| Modificar | `src/styles/global.css` | Adicionar estilos específicos do terminal (scanlines, glow) |

### Parte 2: CLI SSH (novo projeto Go)

| Ação | Arquivo | Responsabilidade |
|------|---------|-----------------|
| Criar | `go.mod` | Módulo Go + dependências |
| Criar | `main.go` | Entry point: servidor SSH com Wish |
| Criar | `tui/model.go` | Model do Bubble Tea (estado, navegação) |
| Criar | `tui/views.go` | Renderização das telas (home, about, skills, etc.) |
| Criar | `tui/styles.go` | Estilos Lip Gloss (cores, bordas, layout) |
| Criar | `tui/content.go` | Dados estáticos (bio, skills, experience, education, contact) |
| Criar | `tui/keys.go` | Keybindings customizadas |
| Criar | `Dockerfile` | Build multi-stage para deploy no fly.io |
| Criar | `fly.toml` | Configuração do fly.io |
| Criar | `.github/workflows/deploy.yml` | CI/CD automático para fly.io |

---

## FASE 1: Site Astro — Aba Terminal + Hint na Hero

### Task 1: Criar componente Terminal.astro

**Files:**
- Create: `src/components/Terminal.astro`

Esta seção será a primeira do site. Mostra uma janela de terminal estilizada com o comando `ssh ssh.talesmiguel.dev` e uma animação de typing. É a "porta de entrada" para devs.

- [ ] **Step 1: Criar o componente Terminal.astro**

```astro
---
// src/components/Terminal.astro
---

<section id="terminal" class="px-6 flex flex-col items-center justify-center bg-bg">
  <div class="w-full max-w-2xl">
    <!-- Terminal window -->
    <div class="terminal-window border border-border rounded-lg overflow-hidden">
      <!-- Title bar -->
      <div class="flex items-center gap-2 px-4 py-3 bg-surface border-b border-border">
        <span class="w-3 h-3 rounded-full bg-[#ff5f57]"></span>
        <span class="w-3 h-3 rounded-full bg-[#febc2e]"></span>
        <span class="w-3 h-3 rounded-full bg-[#28c840]"></span>
        <span class="font-mono text-muted text-xs ml-2">tales@dev ~</span>
      </div>

      <!-- Terminal body -->
      <div class="p-6 font-mono text-sm leading-relaxed bg-bg">
        <div class="text-muted">
          <span class="text-accent">~</span> via <span class="text-accent">ssh</span>
        </div>
        <div class="mt-4">
          <span class="text-accent">$</span>
          <span id="terminal-cmd" class="text-text ml-2"></span>
          <span id="terminal-cursor" class="text-accent">█</span>
        </div>

        <!-- Output that appears after typing -->
        <div id="terminal-output" class="mt-6 hidden">
          <div class="text-muted text-xs mb-4">Connecting to ssh.talesmiguel.dev...</div>
          <pre class="text-accent text-xs leading-tight">
 _____     _             __  __ _                  _
|_   _|_ _| | ___  ___  |  \/  (_) __ _ _   _  ___| |
  | |/ _` | |/ _ \/ __| | |\/| | |/ _` | | | |/ _ \ |
  | | (_| | |  __/\__ \ | |  | | | (_| | |_| |  __/ |
  |_|\__,_|_|\___||___/ |_|  |_|_|\__, |\__,_|\___|_|
                                   |___/
          </pre>
          <div class="mt-4 text-muted text-xs">
            <p>Welcome! Navigate with <span class="text-accent">arrow keys</span> or <span class="text-accent">j/k</span></p>
            <p class="mt-1">
              <span class="text-accent">[about]</span>
              <span class="text-muted ml-2">[skills]</span>
              <span class="text-muted ml-2">[experience]</span>
              <span class="text-muted ml-2">[education]</span>
              <span class="text-muted ml-2">[contact]</span>
            </p>
          </div>
        </div>
      </div>
    </div>

    <!-- CTA below terminal -->
    <p class="text-center text-muted text-sm font-mono mt-6 animate-item">
      Try it yourself:
      <code class="text-accent bg-surface px-2 py-1 rounded ml-1 select-all">ssh ssh.talesmiguel.dev</code>
    </p>
  </div>
</section>

<style>
  .terminal-window {
    box-shadow: 0 0 40px rgba(34, 211, 238, 0.05);
  }
  .animate-item {
    opacity: 0;
    transform: translateY(10px);
    transition: opacity 0.6s ease 1.5s, transform 0.6s ease 1.5s;
  }
  .animate-item.visible {
    opacity: 1;
    transform: translateY(0);
  }
</style>

<script>
  const cmd = "ssh ssh.talesmiguel.dev";
  const el = document.getElementById("terminal-cmd")!;
  const cursor = document.getElementById("terminal-cursor")!;
  const output = document.getElementById("terminal-output")!;
  let i = 0;

  function typeCmd() {
    if (i < cmd.length) {
      el.textContent += cmd[i];
      i++;
      setTimeout(typeCmd, 60 + Math.random() * 40);
    } else {
      // Hide cursor, show output
      setTimeout(() => {
        cursor.style.display = "none";
        output.classList.remove("hidden");

        // Trigger CTA animation
        const cta = document.querySelector("#terminal .animate-item");
        if (cta) cta.classList.add("visible");
      }, 500);
    }
  }

  // Start typing when section is visible
  const section = document.getElementById("terminal")!;
  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((e) => {
        if (e.isIntersecting) {
          setTimeout(typeCmd, 800);
          observer.disconnect();
        }
      });
    },
    { threshold: 0.5 }
  );
  observer.observe(section);
</script>
```

- [ ] **Step 2: Verificar que o site compila**

Run: `cd /home/tales/projects/talesmiguel-dev && npm run build`
Expected: Build sem erros

- [ ] **Step 3: Commit**

```bash
git add src/components/Terminal.astro
git commit -m "feat: add Terminal section component with SSH demo animation"
```

---

### Task 2: Integrar Terminal no index e modificar Hero

**Files:**
- Modify: `src/pages/index.astro` (adicionar import + componente)
- Modify: `src/components/Hero.astro` (adicionar hint com link)

- [ ] **Step 1: Adicionar Terminal ao index.astro como primeira seção**

Em `src/pages/index.astro`, adicionar o import e o componente:

```astro
---
import BaseLayout from '../layouts/BaseLayout.astro';
import Terminal from '../components/Terminal.astro';
import Hero from '../components/Hero.astro';
import Bio from '../components/Bio.astro';
import Skills from '../components/Skills.astro';
import Education from '../components/Education.astro';
import Experience from '../components/Experience.astro';
import Contact from '../components/Contact.astro';
---

<BaseLayout>
  <Terminal />
  <Hero />
  <Bio />
  <Skills />
  <Education />
  <Experience />
  <Contact />
</BaseLayout>
```

- [ ] **Step 2: Adicionar hint na Hero**

Em `src/components/Hero.astro`, adicionar um hint abaixo dos botões existentes (depois do `</div>` que fecha a `flex gap-4`):

```html
    <!-- Depois do div com os botões GitHub e Scroll down -->
    <p class="font-mono text-muted text-xs mt-8 opacity-0" id="dev-hint">
      Hint: if you're a developer, try navigating my site via CLI!
      <a href="#terminal" class="text-accent hover:underline ml-1">See how →</a>
    </p>
```

E no bloco `<script>` existente, adicionar após o typing effect (depois do `setTimeout(type, 400);`):

```javascript
  // Show dev hint after typing completes
  setTimeout(() => {
    const hint = document.getElementById('dev-hint');
    if (hint) {
      hint.style.transition = 'opacity 1s ease';
      hint.style.opacity = '1';
    }
  }, 3000);
```

- [ ] **Step 3: Verificar que o site compila e funciona**

Run: `cd /home/tales/projects/talesmiguel-dev && npm run build`
Expected: Build sem erros

- [ ] **Step 4: Testar localmente**

Run: `cd /home/tales/projects/talesmiguel-dev && npm run dev`
Expected: Navegar para http://localhost:4321, ver a seção Terminal como primeira seção e o hint na Hero

- [ ] **Step 5: Commit**

```bash
git add src/pages/index.astro src/components/Hero.astro
git commit -m "feat: integrate Terminal section and add CLI hint to Hero"
```

---

## FASE 2: CLI SSH em Go

> **🔧 Pré-requisito:** O Tales precisa ter concluído M1 (instalar Go) e M4 (criar repositório) antes de começar esta fase.

### Task 3: Inicializar projeto Go e instalar dependências

**Files:**
- Create: `go.mod` (via `go mod init`)

- [ ] **Step 1: Inicializar módulo Go**

```bash
cd /home/tales/projects/talesmiguel-ssh
go mod init github.com/TalesMiguel/talesmiguel-ssh
```

- [ ] **Step 2: Instalar dependências**

```bash
go get github.com/charmbracelet/wish@latest
go get github.com/charmbracelet/bubbletea@latest
go get github.com/charmbracelet/lipgloss@latest
go get github.com/charmbracelet/ssh@latest
go get github.com/charmbracelet/log@latest
```

- [ ] **Step 3: Verificar go.mod**

Run: `cat go.mod`
Expected: Módulo com as 5 dependências listadas

- [ ] **Step 4: Commit**

```bash
git add go.mod go.sum
git commit -m "chore: init Go module with Charm dependencies"
```

---

### Task 4: Criar estilos Lip Gloss (tui/styles.go)

**Files:**
- Create: `tui/styles.go`

- [ ] **Step 1: Criar o arquivo de estilos**

```go
// tui/styles.go
package tui

import "github.com/charmbracelet/lipgloss"

var (
	// Cores do tema (matching website)
	accent = lipgloss.Color("#22d3ee")
	muted  = lipgloss.Color("#6b7280")
	text   = lipgloss.Color("#e5e5e5")
	bg     = lipgloss.Color("#0a0a0a")
	border = lipgloss.Color("#1f1f1f")

	// Estilos reutilizáveis
	TitleStyle = lipgloss.NewStyle().
			Foreground(accent).
			Bold(true).
			MarginBottom(1)

	SubtitleStyle = lipgloss.NewStyle().
			Foreground(muted).
			MarginBottom(1)

	TextStyle = lipgloss.NewStyle().
			Foreground(text)

	MutedStyle = lipgloss.NewStyle().
			Foreground(muted)

	AccentStyle = lipgloss.NewStyle().
			Foreground(accent)

	ActiveTabStyle = lipgloss.NewStyle().
			Foreground(accent).
			Bold(true).
			Underline(true)

	InactiveTabStyle = lipgloss.NewStyle().
			Foreground(muted)

	BorderStyle = lipgloss.NewStyle().
			Border(lipgloss.RoundedBorder()).
			BorderForeground(border).
			Padding(1, 2)

	HelpStyle = lipgloss.NewStyle().
			Foreground(muted).
			MarginTop(1)
)
```

- [ ] **Step 2: Verificar que compila**

Run: `cd /home/tales/projects/talesmiguel-ssh && go build ./...`
Expected: Sem erros

- [ ] **Step 3: Commit**

```bash
git add tui/styles.go
git commit -m "feat: add Lip Gloss styles matching website theme"
```

---

### Task 5: Criar conteúdo estático (tui/content.go)

**Files:**
- Create: `tui/content.go`

- [ ] **Step 1: Criar o arquivo com todo o conteúdo**

```go
// tui/content.go
package tui

// Dados espelhando o site talesmiguel.dev

var Bio = `I'm a Master's student in Computer Science at Federal University of São Paulo
(UNIFESP), São José dos Campos, researching Artificial Neural Networks, Machine
Learning, and Agentic Systems. I also bring hands-on industry experience as an
ML Developer in the aviation sector and as a full-stack engineer in startups.`

var BioAccent = "When not coding, you'll likely find me behind a drum kit or playing guitar."

type SkillCategory struct {
	Label  string
	Skills []string
}

var Skills = []SkillCategory{
	{Label: "Languages", Skills: []string{"Python", "C", "JavaScript", "TypeScript", "SQL"}},
	{Label: "ML / AI", Skills: []string{"PyTorch", "TensorFlow", "Scikit-learn", "XGBoost", "Data Science"}},
	{Label: "Backend & Web", Skills: []string{"Django", "FastAPI", "Vue.js", "React", "PostgreSQL", "Redis", "MySQL"}},
	{Label: "Infra & Tools", Skills: []string{"Docker", "Git", "Nginx", "Linux", "Microservices", "TDD", "Scrum"}},
}

type ExperienceItem struct {
	Title       string
	Org         string
	Period      string
	Description string
}

var Experience = []ExperienceItem{
	{
		Title:       "Machine Learning Developer",
		Org:         "Embraer",
		Period:      "Sep 2024 – Feb 2025",
		Description: "Failure prediction models for commercial aircraft (E2 jets) using Random Forest, KNN, XGBoost, PyTorch and TensorFlow. Led backend and infrastructure development for an aircraft data platform.",
	},
	{
		Title:       "Full-Stack Web Developer",
		Org:         "TORQI S.A.",
		Period:      "Jan 2023 – May 2023",
		Description: "Full-stack development with focus on backend and system architecture. Contributed to business and architecture decision-making.",
	},
	{
		Title:       "Junior Full-Stack Developer",
		Org:         "Buser",
		Period:      "Nov 2020 – Dec 2022",
		Description: "Built microservices, APIs, and full projects for the customer service platform. Focused on scalability and observability. Mentored interns and delivered internal tech talks.",
	},
}

type EducationItem struct {
	Title       string
	Org         string
	Period      string
	Description string
}

var Education = []EducationItem{
	{
		Title:       "MSc in Computer Science (ongoing)",
		Org:         "UNIFESP — Federal University of São Paulo",
		Period:      "2025 – present",
		Description: "Research in Artificial Neural Networks, Machine Learning, and Agentic Systems.",
	},
	{
		Title:       "BSc in Science and Technology",
		Org:         "UNIFESP — Federal University of São Paulo",
		Period:      "2020 – 2024",
		Description: "Undergraduate degree with focus on Computer Science and Software Engineering.",
	},
	{
		Title:       "BSc in Computer Science (ongoing)",
		Org:         "UNIFESP — Federal University of São Paulo",
		Period:      "2025 – 2027",
	},
}

type ContactLink struct {
	Label string
	Value string
}

var Contact = []ContactLink{
	{Label: "GitHub", Value: "github.com/TalesMiguel"},
	{Label: "LinkedIn", Value: "linkedin.com/in/talesmiguel"},
	{Label: "Email", Value: "tales.miguel@unifesp.br"},
	{Label: "Website", Value: "talesmiguel.dev"},
}
```

- [ ] **Step 2: Verificar que compila**

Run: `cd /home/tales/projects/talesmiguel-ssh && go build ./...`
Expected: Sem erros

- [ ] **Step 3: Commit**

```bash
git add tui/content.go
git commit -m "feat: add static content data mirroring website"
```

---

### Task 6: Criar keybindings (tui/keys.go)

**Files:**
- Create: `tui/keys.go`

- [ ] **Step 1: Criar keybindings**

```go
// tui/keys.go
package tui

import "github.com/charmbracelet/bubbletea"

type keyMap struct{}

func (k keyMap) ShortHelp() []bubbletea.KeyBinding {
	return []bubbletea.KeyBinding{}
}

func (k keyMap) FullHelp() [][]bubbletea.KeyBinding {
	return nil
}

// Key constants para usar no Update
const (
	TabAbout      = 0
	TabSkills     = 1
	TabExperience = 2
	TabEducation  = 3
	TabContact    = 4
)

var TabNames = []string{"about", "skills", "experience", "education", "contact"}
```

- [ ] **Step 2: Verificar que compila**

Run: `cd /home/tales/projects/talesmiguel-ssh && go build ./...`
Expected: Sem erros

- [ ] **Step 3: Commit**

```bash
git add tui/keys.go
git commit -m "feat: add tab constants and key definitions"
```

---

### Task 7: Criar model e views do Bubble Tea (tui/model.go + tui/views.go)

**Files:**
- Create: `tui/model.go`
- Create: `tui/views.go`

- [ ] **Step 1: Criar o model (tui/model.go)**

```go
// tui/model.go
package tui

import (
	"strings"

	tea "github.com/charmbracelet/bubbletea"
)

type Model struct {
	activeTab int
	width     int
	height    int
	quitting  bool
}

func NewModel() Model {
	return Model{
		activeTab: TabAbout,
	}
}

func (m Model) Init() tea.Cmd {
	return nil
}

func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.height = msg.Height
		return m, nil

	case tea.KeyMsg:
		switch msg.String() {
		case "q", "ctrl+c", "esc":
			m.quitting = true
			return m, tea.Quit

		case "tab", "l", "right":
			m.activeTab = (m.activeTab + 1) % len(TabNames)
			return m, nil

		case "shift+tab", "h", "left":
			m.activeTab = (m.activeTab - 1 + len(TabNames)) % len(TabNames)
			return m, nil

		case "1":
			m.activeTab = TabAbout
		case "2":
			m.activeTab = TabSkills
		case "3":
			m.activeTab = TabExperience
		case "4":
			m.activeTab = TabEducation
		case "5":
			m.activeTab = TabContact
		}
	}

	return m, nil
}

func (m Model) View() string {
	if m.quitting {
		return AccentStyle.Render("Thanks for visiting! → talesmiguel.dev") + "\n"
	}

	var b strings.Builder

	// Header / ASCII art
	b.WriteString(renderHeader())
	b.WriteString("\n")

	// Tabs
	b.WriteString(renderTabs(m.activeTab))
	b.WriteString("\n\n")

	// Content
	b.WriteString(renderContent(m.activeTab, m.width))
	b.WriteString("\n\n")

	// Help
	b.WriteString(HelpStyle.Render("←/→ navigate • 1-5 jump • q quit"))
	b.WriteString("\n")

	return b.String()
}
```

- [ ] **Step 2: Criar as views (tui/views.go)**

```go
// tui/views.go
package tui

import (
	"fmt"
	"strings"
)

const asciiArt = `
 _____     _             __  __ _                  _
|_   _|_ _| | ___  ___  |  \/  (_) __ _ _   _  ___| |
  | |/ _` + "`" + ` | |/ _ \/ __| | |\/| | |/ _` + "`" + ` | | | |/ _ \ |
  | | (_| | |  __/\__ \ | |  | | | (_| | |_| |  __/ |
  |_|\__,_|_|\___||___/ |_|  |_|_|\__, |\__,_|\___|_|
                                   |___/`

func renderHeader() string {
	return AccentStyle.Render(asciiArt)
}

func renderTabs(active int) string {
	var tabs []string
	for i, name := range TabNames {
		if i == active {
			tabs = append(tabs, ActiveTabStyle.Render("["+name+"]"))
		} else {
			tabs = append(tabs, InactiveTabStyle.Render("["+name+"]"))
		}
	}
	return strings.Join(tabs, "  ")
}

func renderContent(tab int, width int) string {
	maxWidth := width - 4
	if maxWidth < 40 {
		maxWidth = 60
	}
	if maxWidth > 80 {
		maxWidth = 80
	}

	switch tab {
	case TabAbout:
		return renderAbout(maxWidth)
	case TabSkills:
		return renderSkills()
	case TabExperience:
		return renderExperience(maxWidth)
	case TabEducation:
		return renderEducation(maxWidth)
	case TabContact:
		return renderContact()
	default:
		return ""
	}
}

func renderAbout(width int) string {
	var b strings.Builder
	b.WriteString(TitleStyle.Render("About"))
	b.WriteString("\n\n")
	b.WriteString(TextStyle.Width(width).Render(Bio))
	b.WriteString("\n\n")
	b.WriteString(AccentStyle.Render("  │ " + BioAccent))
	return b.String()
}

func renderSkills() string {
	var b strings.Builder
	b.WriteString(TitleStyle.Render("Skills"))
	b.WriteString("\n")

	for _, cat := range Skills {
		b.WriteString("\n")
		b.WriteString(SubtitleStyle.Render("  " + cat.Label))
		b.WriteString("\n")

		var pills []string
		for _, skill := range cat.Skills {
			pills = append(pills, AccentStyle.Render("["+skill+"]"))
		}
		b.WriteString("  " + strings.Join(pills, " "))
		b.WriteString("\n")
	}
	return b.String()
}

func renderExperience(width int) string {
	var b strings.Builder
	b.WriteString(TitleStyle.Render("Experience"))
	b.WriteString("\n")

	for _, item := range Experience {
		b.WriteString("\n")
		b.WriteString(AccentStyle.Render("  " + item.Period))
		b.WriteString("\n")
		b.WriteString(TextStyle.Bold(true).Render("  " + item.Title))
		b.WriteString("\n")
		b.WriteString(MutedStyle.Render("  " + item.Org))
		b.WriteString("\n")
		b.WriteString(MutedStyle.Width(width - 4).Render("  " + item.Description))
		b.WriteString("\n")
	}
	return b.String()
}

func renderEducation(width int) string {
	var b strings.Builder
	b.WriteString(TitleStyle.Render("Education"))
	b.WriteString("\n")

	for _, item := range Education {
		b.WriteString("\n")
		b.WriteString(AccentStyle.Render("  " + item.Period))
		b.WriteString("\n")
		b.WriteString(TextStyle.Bold(true).Render("  " + item.Title))
		b.WriteString("\n")
		b.WriteString(MutedStyle.Render("  " + item.Org))
		if item.Description != "" {
			b.WriteString("\n")
			b.WriteString(MutedStyle.Width(width - 4).Render("  " + item.Description))
		}
		b.WriteString("\n")
	}
	return b.String()
}

func renderContact() string {
	var b strings.Builder
	b.WriteString(TitleStyle.Render("Contact"))
	b.WriteString("\n\n")
	b.WriteString(TextStyle.Render("  Let's connect."))
	b.WriteString("\n\n")

	for _, link := range Contact {
		b.WriteString(fmt.Sprintf("  %s  %s\n",
			MutedStyle.Width(10).Render(link.Label),
			AccentStyle.Render(link.Value),
		))
	}
	return b.String()
}
```

- [ ] **Step 3: Verificar que compila**

Run: `cd /home/tales/projects/talesmiguel-ssh && go build ./...`
Expected: Sem erros

- [ ] **Step 4: Commit**

```bash
git add tui/model.go tui/views.go
git commit -m "feat: add Bubble Tea model and views for all tabs"
```

---

### Task 8: Criar o servidor SSH (main.go)

**Files:**
- Create: `main.go`

- [ ] **Step 1: Criar main.go com servidor Wish**

```go
// main.go
package main

import (
	"context"
	"errors"
	"fmt"
	"net"
	"os"
	"os/signal"
	"syscall"
	"time"

	tea "github.com/charmbracelet/bubbletea"
	"github.com/charmbracelet/log"
	"github.com/charmbracelet/ssh"
	"github.com/charmbracelet/wish"
	"github.com/charmbracelet/wish/bubbletea"

	"github.com/TalesMiguel/talesmiguel-ssh/tui"
)

const (
	defaultHost = "0.0.0.0"
	defaultPort = 23234
)

func main() {
	host := defaultHost
	port := defaultPort

	if p := os.Getenv("PORT"); p != "" {
		fmt.Sscanf(p, "%d", &port)
	}

	s, err := wish.NewServer(
		wish.WithAddress(net.JoinHostPort(host, fmt.Sprintf("%d", port))),
		wish.WithHostKeyPath(".ssh/id_ed25519"),
		wish.WithMiddleware(
			bubbletea.Middleware(teaHandler),
			activeTerminal(),
		),
	)
	if err != nil {
		log.Error("Could not start server", "error", err)
		os.Exit(1)
	}

	done := make(chan os.Signal, 1)
	signal.Notify(done, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)

	log.Info("Starting SSH server", "host", host, "port", port)
	go func() {
		if err := s.ListenAndServe(); err != nil && !errors.Is(err, ssh.ErrServerClosed) {
			log.Error("Could not start server", "error", err)
			done <- nil
		}
	}()

	<-done
	log.Info("Stopping SSH server")
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	if err := s.Shutdown(ctx); err != nil && !errors.Is(err, ssh.ErrServerClosed) {
		log.Error("Could not stop server", "error", err)
	}
}

func teaHandler(s ssh.Session) (tea.Model, []tea.ProgramOption) {
	return tui.NewModel(), []tea.ProgramOption{tea.WithAltScreen()}
}

// activeTerminal rejects non-interactive sessions (no PTY)
func activeTerminal() wish.Middleware {
	return func(next ssh.Handler) ssh.Handler {
		return func(s ssh.Session) {
			_, _, active := s.Pty()
			if !active {
				wish.Fatalln(s, fmt.Errorf("no active terminal, please use: ssh -t ssh.talesmiguel.dev"))
				return
			}
			next(s)
		}
	}
}
```

- [ ] **Step 2: Verificar que compila**

Run: `cd /home/tales/projects/talesmiguel-ssh && go build -o talesmiguel-ssh .`
Expected: Binário `talesmiguel-ssh` criado sem erros

- [ ] **Step 3: Testar localmente**

```bash
# Terminal 1: rodar o servidor
mkdir -p .ssh
cd /home/tales/projects/talesmiguel-ssh && ./talesmiguel-ssh

# Terminal 2: conectar via SSH
ssh -p 23234 localhost
```

Expected: Ver o ASCII art, tabs navegáveis, conteúdo renderizado

- [ ] **Step 4: Commit**

```bash
git add main.go
echo ".ssh/" >> .gitignore
echo "talesmiguel-ssh" >> .gitignore
git add .gitignore
git commit -m "feat: add Wish SSH server entry point"
```

---

## FASE 3: Deploy no fly.io

> **🔧 Pré-requisito:** O Tales precisa ter concluído M2 (conta fly.io + flyctl) antes desta fase.

### Task 9: Criar Dockerfile e fly.toml

**Files:**
- Create: `Dockerfile`
- Create: `fly.toml`

- [ ] **Step 1: Criar Dockerfile multi-stage**

```dockerfile
# Dockerfile
FROM golang:1.24-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /talesmiguel-ssh .

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /talesmiguel-ssh /talesmiguel-ssh

EXPOSE 23234
CMD ["/talesmiguel-ssh"]
```

- [ ] **Step 2: Criar fly.toml**

```toml
# fly.toml
app = "talesmiguel-ssh"
primary_region = "gru"  # São Paulo — mais perto do Tales e visitantes BR

[build]
  dockerfile = "Dockerfile"

[[services]]
  internal_port = 23234
  protocol = "tcp"

  [[services.ports]]
    port = 22
```

- [ ] **Step 3: Commit**

```bash
git add Dockerfile fly.toml
git commit -m "chore: add Dockerfile and fly.toml for deployment"
```

---

### Task 10: Deploy inicial no fly.io

> **🔧 Esta task é executada pelo Tales (precisa de autenticação fly.io)**

- [ ] **Step 1: Criar app no fly.io**

```bash
cd /home/tales/projects/talesmiguel-ssh
flyctl apps create talesmiguel-ssh
```

> Se o nome `talesmiguel-ssh` já estiver em uso, escolha outro e atualize o `fly.toml`.

- [ ] **Step 2: Deploy**

```bash
flyctl deploy
```

Expected: Build e deploy com sucesso. O fly.io vai mostrar o hostname (ex: `talesmiguel-ssh.fly.dev`)

- [ ] **Step 3: Alocar IP público (necessário para SSH direto)**

```bash
flyctl ips allocate-v4 --shared
flyctl ips allocate-v6
```

Expected: IPs alocados. Anotar o IPv4 para configurar o DNS.

- [ ] **Step 4: Testar conexão**

```bash
ssh talesmiguel-ssh.fly.dev
```

Expected: Conectar ao TUI via SSH

- [ ] **Step 5: Commit (se houve ajustes)**

```bash
git add -A
git commit -m "chore: finalize fly.io deployment config"
```

---

### Task 11: Configurar DNS e testar domínio customizado

> **🔧 Esta task é 100% manual do Tales**

- [ ] **Step 1: Adicionar certificado no fly.io**

```bash
flyctl certs create ssh.talesmiguel.dev
```

- [ ] **Step 2: Configurar DNS (ação M3)**

Seguir as instruções de M3 acima — adicionar registro A no painel DNS apontando `ssh` para o IP do fly.io.

- [ ] **Step 3: Aguardar propagação DNS (5-30 min)**

```bash
# Verificar propagação
dig ssh.talesmiguel.dev
```

- [ ] **Step 4: Testar com domínio customizado**

```bash
ssh ssh.talesmiguel.dev
```

Expected: Conectar ao TUI com sucesso

---

### Task 12: CI/CD automático (opcional)

**Files:**
- Create: `.github/workflows/deploy.yml`

- [ ] **Step 1: Criar workflow de deploy**

```yaml
# .github/workflows/deploy.yml
name: Deploy to fly.io

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: superfly/flyctl-actions/setup-flyctl@master

      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

- [ ] **Step 2: Gerar token fly.io (🔧 manual)**

```bash
flyctl tokens create deploy -x 999999h
```

Copiar o token e adicionar como secret no GitHub:
1. Ir em github.com/TalesMiguel/talesmiguel-ssh → Settings → Secrets → Actions
2. Criar secret `FLY_API_TOKEN` com o valor do token

- [ ] **Step 3: Commit e testar**

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add automatic deploy to fly.io on push"
git push
```

Expected: GitHub Action roda e deploya automaticamente

---

## Resumo de responsabilidades

| # | O que | Quem faz | Quando |
|---|-------|----------|--------|
| M1 | Instalar Go | Tales | Antes da Fase 2 |
| M2 | Criar conta fly.io + instalar flyctl | Tales | Antes da Fase 3 |
| M3 | Configurar DNS (ssh.talesmiguel.dev) | Tales | Depois da Task 10 |
| M4 | Criar repositório Git | Tales | Antes da Fase 2 |
| T1 | Componente Terminal.astro | Claude | Fase 1 |
| T2 | Integrar Terminal + hint Hero | Claude | Fase 1 |
| T3 | Init projeto Go | Claude | Fase 2 |
| T4 | Estilos Lip Gloss | Claude | Fase 2 |
| T5 | Conteúdo estático | Claude | Fase 2 |
| T6 | Keybindings | Claude | Fase 2 |
| T7 | Model + Views Bubble Tea | Claude | Fase 2 |
| T8 | Servidor SSH (main.go) | Claude | Fase 2 |
| T9 | Dockerfile + fly.toml | Claude | Fase 3 |
| T10 | Deploy no fly.io | Tales | Fase 3 |
| T11 | DNS + domínio | Tales | Fase 3 |
| T12 | CI/CD (opcional) | Tales + Claude | Fase 3 |
