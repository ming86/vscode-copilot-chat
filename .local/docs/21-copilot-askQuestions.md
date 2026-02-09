# Copilot askQuestions Tool Documentation

## Overview

The `askQuestions` tool enables the language model to request clarifications, preferences, or confirmations from the user during task execution. Questions are presented via VS Code's native QuickPick UI, supporting both single-select and multi-select modes with automatic free-text input capability.

**Primary Use Cases:**
- Clarifying ambiguous requirements before proceeding
- Gathering user preferences on implementation choices
- Confirming high-impact decisions before execution
- Collecting missing information without guessing

**Tool Category:** VS Code Interaction

**UI Mechanism:** VS Code QuickPick API with optional InputBox for free-text

---

## Complete Implementation

This section provides the full, copy-paste-ready implementation.

### Dependencies

```typescript
// VS Code API (stable, available in VS Code 1.93+)
import * as vscode from 'vscode';

// For standalone extension, replace these with:
// - ITelemetryService → @vscode/extension-telemetry or no-op
// - ILogService → vscode.window.createOutputChannel() or console
// - DisposableStore → simple array of disposables or vscode.Disposable.from()
// - StopWatch → performance.now() inline
// - CancellationToken → vscode.CancellationToken
```

### Type Definitions

```typescript
/**
 * A single option within a question
 */
interface IQuestionOption {
	/** Option label text */
	label: string;
	/** Optional description shown below label */
	description?: string;
	/** Mark as recommended (displays star icon) */
	recommended?: boolean;
}

/**
 * A question to present to the user
 */
interface IQuestion {
	/** Short label (max 12 chars), used as unique identifier and QuickPick title */
	header: string;
	/** Complete question text displayed as placeholder */
	question: string;
	/** Allow multiple selections (default: false) */
	multiSelect?: boolean;
	/** 2-4 options for the user to choose from */
	options: IQuestionOption[];
}

/**
 * Tool input parameters
 */
interface IAskQuestionsParams {
	/** Array of 1-4 questions to present sequentially */
	questions: IQuestion[];
}

/**
 * Tool output returned to the model
 */
interface IAnswerResult {
	/** Answers keyed by question header */
	answers: Record<string, {
		/** Array of selected option labels */
		selected: string[];
		/** Custom text if "Other..." was used, null otherwise */
		freeText: string | null;
		/** True if user cancelled/escaped */
		skipped: boolean;
	}>;
}

/**
 * Extended QuickPickItem for internal use
 */
interface IQuickPickOptionItem extends vscode.QuickPickItem {
	/** Whether this is the recommended option */
	isRecommended?: boolean;
	/** Whether this is the "Other..." free-text option */
	isFreeText?: boolean;
	/** Original label without star prefix (used in results) */
	originalLabel: string;
}
```

### Complete Tool Class

```typescript
export class AskQuestionsTool implements vscode.LanguageModelTool<IAskQuestionsParams> {
	/**
	 * Tool name for registration
	 * Use with vscode.lm.registerTool('copilot_askQuestions', new AskQuestionsTool())
	 */
	public static readonly toolName = 'ask_questions';

	constructor(
		// Replace with your telemetry service or remove
		private readonly _telemetryService?: { sendTelemetryEvent: (name: string, props: Record<string, string | undefined>, metrics: Record<string, number>) => void },
		// Replace with vscode.window.createOutputChannel() or remove
		private readonly _logService?: { trace: (msg: string) => void },
	) { }

	/**
	 * Main invocation method called by the language model
	 */
	async invoke(
		options: vscode.LanguageModelToolInvocationOptions<IAskQuestionsParams>,
		token: vscode.CancellationToken
	): Promise<vscode.LanguageModelToolResult> {
		const startTime = performance.now();
		const { questions } = options.input;
		this._logService?.trace(`[AskQuestionsTool] Invoking with ${questions.length} question(s)`);

		const result: IAnswerResult = { answers: {} };
		let currentStep = 0;

		// Sequential question loop with back-navigation support
		while (currentStep < questions.length) {
			// Check cancellation before each question
			if (token.isCancellationRequested) {
				// Mark remaining questions as skipped
				for (let i = currentStep; i < questions.length; i++) {
					const q = questions[i];
					result.answers[q.header] = {
						selected: [],
						freeText: null,
						skipped: true
					};
				}
				break;
			}

			const question = questions[currentStep];
			const answer = await this._askQuestion(question, currentStep, questions.length, token);

			if (answer === 'back' && currentStep > 0) {
				// Go back to previous question
				currentStep--;
				continue;
			}

			if (answer === 'skipped') {
				// User pressed ESC - mark current and remaining questions as skipped
				for (let i = currentStep; i < questions.length; i++) {
					const q = questions[i];
					result.answers[q.header] = {
						selected: [],
						freeText: null,
						skipped: true
					};
				}
				break;
			}

			// Store answer and advance
			result.answers[question.header] = answer as { selected: string[]; freeText: string | null; skipped: boolean };
			currentStep++;
		}

		// Calculate and send telemetry
		const duration = performance.now() - startTime;
		this._sendTelemetry(options.toolInvocationToken, questions, result, duration);

		// Return JSON result to model
		return new vscode.LanguageModelToolResult([
			new vscode.LanguageModelTextPart(JSON.stringify(result))
		]);
	}

	/**
	 * Validation and UI message preparation
	 */
	prepareInvocation(
		options: vscode.LanguageModelToolInvocationPrepareOptions<IAskQuestionsParams>,
		_token: vscode.CancellationToken
	): vscode.ProviderResult<vscode.PreparedToolInvocation> {
		const { questions } = options.input;

		// Validate: at least one question required
		if (!questions || questions.length === 0) {
			throw new Error(vscode.l10n.t('No questions provided. The questions array must contain at least one question.'));
		}

		// Validate: each question must have at least 2 options
		for (const question of questions) {
			if (!question.options || question.options.length < 2) {
				throw new Error(vscode.l10n.t('Question "{0}" must have at least two options.', question.header));
			}
		}

		// Build UI messages
		const questionCount = questions.length;
		const message = questionCount === 1
			? vscode.l10n.t('Asking a question')
			: vscode.l10n.t('Asking {0} questions', questionCount);
		const pastMessage = questionCount === 1
			? vscode.l10n.t('Asked a question')
			: vscode.l10n.t('Asked {0} questions', questionCount);

		return {
			invocationMessage: new vscode.MarkdownString(message),
			pastTenseMessage: new vscode.MarkdownString(pastMessage)
		};
	}

	/**
	 * Get the default option (recommended or first)
	 */
	private _getDefaultOption(question: IQuestion): IQuestionOption {
		const recommended = question.options.find(opt => opt.recommended);
		return recommended ?? question.options[0];
	}

	/**
	 * Present a single question via QuickPick
	 */
	private async _askQuestion(
		question: IQuestion,
		step: number,
		totalSteps: number,
		token: vscode.CancellationToken
	): Promise<{ selected: string[]; freeText: string | null; skipped: boolean } | 'back' | 'skipped'> {
		// Check cancellation before showing UI
		if (token.isCancellationRequested) {
			return 'skipped';
		}

		return new Promise((resolve) => {
			// Race condition protection: prevent double-resolution
			let resolved = false;
			const safeResolve = (value: { selected: string[]; freeText: string | null; skipped: boolean } | 'back' | 'skipped') => {
				if (!resolved) {
					resolved = true;
					resolve(value);
				}
			};

			// Create QuickPick
			const quickPick = vscode.window.createQuickPick<IQuickPickOptionItem>();
			quickPick.title = question.header;
			quickPick.placeholder = question.question;
			quickPick.step = step + 1;
			quickPick.totalSteps = totalSteps;
			quickPick.canSelectMany = question.multiSelect ?? false;
			quickPick.ignoreFocusOut = true;

			// Build option items
			const items: IQuickPickOptionItem[] = question.options.map(opt => ({
				label: opt.recommended ? `$(star-full) ${opt.label}` : opt.label,
				description: opt.description,
				isRecommended: opt.recommended,
				isFreeText: false,
				originalLabel: opt.label
			}));

			// Always add "Other..." free-text option
			items.push({
				label: vscode.l10n.t('Other...'),
				description: vscode.l10n.t('Enter custom answer'),
				isFreeText: true,
				originalLabel: 'Other'
			});

			quickPick.items = items;

			// Set default selection
			const defaultOption = this._getDefaultOption(question);
			const defaultItem = items.find(item =>
				item.originalLabel === defaultOption.label || item.isRecommended
			);

			if (defaultItem) {
				if (question.multiSelect) {
					quickPick.selectedItems = [defaultItem]; // Pre-checked for multi-select
				} else {
					quickPick.activeItems = [defaultItem];   // Pre-highlighted for single-select
				}
			}

			// Add back button for multi-step flows
			if (step > 0) {
				quickPick.buttons = [vscode.QuickInputButtons.Back];
			}

			// Disposables for cleanup
			const disposables: vscode.Disposable[] = [quickPick];

			// Handle cancellation token
			disposables.push(
				token.onCancellationRequested(() => {
					quickPick.hide();
				})
			);

			// Handle back button
			disposables.push(
				quickPick.onDidTriggerButton(button => {
					if (button === vscode.QuickInputButtons.Back) {
						quickPick.hide();
						safeResolve('back');
					}
				})
			);

			// Handle selection acceptance
			disposables.push(
				quickPick.onDidAccept(async () => {
					const selectedItems = question.multiSelect
						? quickPick.selectedItems
						: quickPick.activeItems;

					// No selection: use default
					if (selectedItems.length === 0) {
						quickPick.hide();
						safeResolve({
							selected: [defaultOption.label],
							freeText: null,
							skipped: false
						});
						return;
					}

					// Check if "Other..." was selected
					const freeTextItem = selectedItems.find(item => item.isFreeText);
					if (freeTextItem) {
						// Mark resolved before hiding to prevent onDidHide race
						resolved = true;
						quickPick.hide();

						// Show input box for custom text
						const freeTextInput = await vscode.window.showInputBox({
							prompt: question.question,
							placeHolder: vscode.l10n.t('Enter your answer'),
							ignoreFocusOut: true
						}, token);

						// Get other selections (excluding "Other...")
						const otherSelections = selectedItems
							.filter(item => !item.isFreeText)
							.map(item => item.originalLabel);

						if (freeTextInput === undefined) {
							// User cancelled input box
							if (otherSelections.length > 0) {
								// Preserve other selections
								resolve({
									selected: otherSelections,
									freeText: null,
									skipped: false
								});
							} else {
								// No other selections: treat as skipped
								resolve('skipped');
							}
						} else {
							// Free text provided
							resolve({
								selected: otherSelections.length > 0 ? otherSelections : [freeTextItem.originalLabel],
								freeText: freeTextInput,
								skipped: false
							});
						}
						return;
					}

					// Regular selection (no free text)
					quickPick.hide();
					safeResolve({
						selected: selectedItems.map(item => item.originalLabel),
						freeText: null,
						skipped: false
					});
				})
			);

			// Handle hide (ESC or dismiss)
			disposables.push(
				quickPick.onDidHide(() => {
					// Resolve before disposal to prevent race conditions
					safeResolve('skipped');
					disposables.forEach(d => d.dispose());
				})
			);

			// Show the QuickPick
			quickPick.show();
		});
	}

	/**
	 * Send telemetry metrics
	 */
	private _sendTelemetry(
		requestId: string | undefined,
		questions: IQuestion[],
		result: IAnswerResult,
		duration: number
	): void {
		if (!this._telemetryService) {
			return;
		}

		const answers = Object.values(result.answers);
		const answeredCount = answers.filter(a => !a.skipped).length;
		const skippedCount = answers.filter(a => a.skipped).length;
		const freeTextCount = answers.filter(a => a.freeText !== null).length;
		const recommendedAvailableCount = questions.filter(q => q.options.some(opt => opt.recommended)).length;
		const recommendedSelectedCount = questions.filter(q => {
			const answer = result.answers[q.header];
			const recommendedOption = q.options.find(opt => opt.recommended);
			return answer && !answer.skipped && recommendedOption && answer.selected.includes(recommendedOption.label);
		}).length;

		this._telemetryService.sendTelemetryEvent(
			'askQuestionsToolInvoked',
			{ requestId },
			{
				questionCount: questions.length,
				answeredCount,
				skippedCount,
				freeTextCount,
				recommendedAvailableCount,
				recommendedSelectedCount,
				duration
			}
		);
	}
}
```

### Extension Registration (activate function)

```typescript
export function activate(context: vscode.ExtensionContext) {
	// Register the tool
	const tool = new AskQuestionsTool(
		// Optional: pass telemetry service
		// Optional: pass log service
	);

	context.subscriptions.push(
		vscode.lm.registerTool('copilot_askQuestions', tool)
	);
}
```

### package.json Contribution (Complete)

```json
{
	"contributes": {
		"languageModelTools": [
			{
				"name": "copilot_askQuestions",
				"toolReferenceName": "askQuestions",
				"displayName": "Ask Questions",
				"userDescription": "Ask questions to clarify requirements before proceeding with a task.",
				"modelDescription": "Ask the user questions when progress is blocked or a decision carries significant risk. Prefer proposing a sensible default so users can confirm quickly.\n\nWhen to use:\n- Clarify ambiguous requirements before proceeding\n- Get user preferences on implementation choices\n- Confirm high-impact decisions\n\nQuestion guidelines:\n- Batch related questions into a single call to minimize interruption\n- Provide brief context explaining what is being decided and why it matters\n- Mark one option as `recommended` with a short justification\n- Keep options mutually exclusive for single-select; use `multiSelect: true` only when choices are additive\n\nAfter receiving answers:\n- Restate the chosen decisions briefly and proceed\n- Do not re-ask unless requirements change\n\n- An \"Other\" option is automatically shown to users for custom input—do not add your own \"Something else\" or similar option.",
				"icon": "$(question)",
				"canBeReferencedInPrompt": true,
				"when": "config.myExtension.askQuestions.enabled",
				"inputSchema": {
					"type": "object",
					"properties": {
						"questions": {
							"type": "array",
							"description": "Array of 1-4 questions to ask the user",
							"minItems": 1,
							"maxItems": 4,
							"items": {
								"type": "object",
								"properties": {
									"header": {
										"type": "string",
										"description": "A short label (max 12 chars) displayed as a quick pick header, also used as the unique identifier for the question",
										"maxLength": 12
									},
									"question": {
										"type": "string",
										"description": "The complete question text to display"
									},
									"multiSelect": {
										"type": "boolean",
										"description": "Allow multiple selections",
										"default": false
									},
									"options": {
										"type": "array",
										"description": "2-4 options for the user to choose from",
										"minItems": 2,
										"maxItems": 4,
										"items": {
											"type": "object",
											"properties": {
												"label": {
													"type": "string",
													"description": "Option label text"
												},
												"description": {
													"type": "string",
													"description": "Optional description for the option"
												},
												"recommended": {
													"type": "boolean",
													"description": "Mark this option as recommended"
												}
											},
											"required": ["label"]
										}
									}
								},
								"required": ["header", "question", "options"]
							}
						}
					},
					"required": ["questions"]
				}
			}
		],
		"configuration": {
			"type": "object",
			"title": "Ask Questions Tool",
			"properties": {
				"myExtension.askQuestions.enabled": {
					"type": "boolean",
					"default": true,
					"description": "Allow agent mode to ask clarifying questions before proceeding with a task."
				}
			}
		}
	}
}
```

---

## API Reference (Quick Summary)

The following sections provide additional reference documentation. The complete implementation above is the authoritative source.

### Tool Declaration (JSON Schema)

```typescript
{
	name: 'copilot_askQuestions',
	toolReferenceName: 'askQuestions',
	displayName: 'Ask Questions',
	icon: '$(question)',
	canBeReferencedInPrompt: true,
	when: 'config.github.copilot.chat.askQuestions.enabled',
	inputSchema: {
		type: 'object',
		properties: {
			questions: {
				type: 'array',
				description: 'Array of 1-4 questions to ask the user',
				minItems: 1,
				maxItems: 4,
				items: {
					type: 'object',
					properties: {
						header: {
							type: 'string',
							description: 'A short label (max 12 chars) displayed as a quick pick header, also used as the unique identifier for the question',
							maxLength: 12
						},
						question: {
							type: 'string',
							description: 'The complete question text to display'
						},
						multiSelect: {
							type: 'boolean',
							description: 'Allow multiple selections',
							default: false
						},
						options: {
							type: 'array',
							description: '2-4 options for the user to choose from',
							minItems: 2,
							maxItems: 4,
							items: {
								type: 'object',
								properties: {
									label: { type: 'string', description: 'Option label text' },
									description: { type: 'string', description: 'Optional description for the option' },
									recommended: { type: 'boolean', description: 'Mark this option as recommended' }
								},
								required: ['label']
							}
						}
					},
					required: ['header', 'question', 'options']
				}
			}
		},
		required: ['questions']
	}
}
```

## Input Parameters

### questions (required)
- **Type:** `IQuestion[]`
- **Constraints:** 1-4 items
- **Description:** Array of questions to present to the user sequentially

### Question Object Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `header` | `string` | Yes | Short label (max 12 chars), used as unique identifier and QuickPick title |
| `question` | `string` | Yes | Complete question text displayed as placeholder |
| `multiSelect` | `boolean` | No | Allow multiple selections (default: `false`) |
| `options` | `IQuestionOption[]` | Yes | 2-4 options for the user to choose from |

### Option Object Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `label` | `string` | Yes | Option label text |
| `description` | `string` | No | Additional description shown below label |
| `recommended` | `boolean` | No | Mark as recommended (displays star icon) |

### Examples

```typescript
// Single question with recommended option
{
	questions: [{
		header: 'Language',
		question: 'Which programming language should I use for this project?',
		options: [
			{ label: 'TypeScript', description: 'Strongly typed JavaScript', recommended: true },
			{ label: 'JavaScript', description: 'Dynamic scripting language' },
			{ label: 'Python', description: 'General purpose language' }
		]
	}]
}

// Multi-select question
{
	questions: [{
		header: 'Features',
		question: 'Which features should I include?',
		multiSelect: true,
		options: [
			{ label: 'Logging', recommended: true },
			{ label: 'Error handling' },
			{ label: 'Unit tests' },
			{ label: 'Documentation' }
		]
	}]
}

// Multiple questions
{
	questions: [
		{
			header: 'Framework',
			question: 'Which framework do you prefer?',
			options: [
				{ label: 'React', recommended: true },
				{ label: 'Vue' },
				{ label: 'Angular' }
			]
		},
		{
			header: 'Styling',
			question: 'How should styles be managed?',
			options: [
				{ label: 'CSS Modules' },
				{ label: 'Tailwind CSS', recommended: true },
				{ label: 'Styled Components' }
			]
		}
	]
}
```

## Output Format

The tool returns a JSON object with answers keyed by the question's `header`:

```typescript
interface IAnswerResult {
	answers: Record<string, {
		selected: string[];      // Array of selected option labels
		freeText: string | null; // Custom text if "Other..." was used
		skipped: boolean;        // True if user cancelled/escaped
	}>;
}
```

### Example Output

**Single selection:**
```json
{
	"answers": {
		"Language": {
			"selected": ["TypeScript"],
			"freeText": null,
			"skipped": false
		}
	}
}
```

**Multi-select with free text:**
```json
{
	"answers": {
		"Features": {
			"selected": ["Logging", "Error handling", "Other"],
			"freeText": "Custom metrics collection",
			"skipped": false
		}
	}
}
```

**User cancelled:**
```json
{
	"answers": {
		"Language": {
			"selected": [],
			"freeText": null,
			"skipped": true
		}
	}
}
```

## Architecture

### High-Level Flow

```
Model Request → AskQuestionsTool.invoke()
    ↓
prepareInvocation() [Validation]
    ↓
Sequential Question Loop
    ↓
┌─────────────────────────────────────┐
│ For each question:                   │
│   1. Create QuickPick UI             │
│   2. Render options + "Other..."     │
│   3. Wait for selection              │
│   4. Handle free-text if needed      │
│   5. Store answer, advance or back   │
└─────────────────────────────────────┘
    ↓
Aggregate Answers → JSON Result
    ↓
LanguageModelToolResult
```

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    AskQuestionsTool                              │
│  File: src/extension/tools/vscode-node/askQuestionsTool.ts      │
├─────────────────────────────────────────────────────────────────┤
│ Static:                                                          │
│   static readonly toolName = ToolName.AskQuestions              │
│                                                                  │
│ Dependencies:                                                    │
│   ITelemetryService - Usage metrics                             │
│   ILogService - Debug logging                                   │
│                                                                  │
│ Methods:                                                         │
│   prepareInvocation() - Validation, UI message generation       │
│   invoke() - Main execution loop                                │
│   private askQuestion() - Single question QuickPick handler     │
│   private _getDefaultOption() - Recommended option resolution   │
│   private _sendTelemetry() - Metrics emission                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VS Code QuickPick API                         │
├─────────────────────────────────────────────────────────────────┤
│ vscode.window.createQuickPick<IQuickPickOptionItem>()           │
│   - title: question.header                                      │
│   - placeholder: question.question                              │
│   - canSelectMany: question.multiSelect                         │
│   - ignoreFocusOut: true                                        │
│   - step/totalSteps: Navigation indicators                      │
│   - buttons: [Back] when step > 0                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    VS Code InputBox API                          │
├─────────────────────────────────────────────────────────────────┤
│ vscode.window.showInputBox() — for "Other..." free text         │
│   - prompt: question.question                                   │
│   - placeHolder: "Enter your answer"                            │
│   - ignoreFocusOut: true                                        │
└─────────────────────────────────────────────────────────────────┘
```

## Key Implementation Details

### Tool Registration

```typescript
// src/extension/tools/vscode-node/askQuestionsTool.ts
ToolRegistry.registerTool(AskQuestionsTool);
```

The tool self-registers at module load time via the `ToolRegistry` pattern.

### Tool Name Mapping

| Registry | Value |
|----------|-------|
| `ToolName.AskQuestions` | `'ask_questions'` |
| `ContributedToolName.AskQuestions` | `'copilot_askQuestions'` |

### Category Assignment

```typescript
toolCategory = ToolCategory.VSCodeInteraction
```

### Validation (prepareInvocation)

Runtime validation occurs before any UI is shown:

```typescript
prepareInvocation(options, token) {
	const { questions } = options.input;

	if (!questions || questions.length === 0) {
		throw new Error('No questions provided. The questions array must contain at least one question.');
	}

	for (const question of questions) {
		if (!question.options || question.options.length < 2) {
			throw new Error(`Question "${question.header}" must have at least two options.`);
		}
	}

	// Singular/plural message formatting
	const questionCount = questions.length;
	const message = questionCount === 1
		? vscode.l10n.t('Asking a question')
		: vscode.l10n.t('Asking {0} questions', questionCount);
	const pastMessage = questionCount === 1
		? vscode.l10n.t('Asked a question')
		: vscode.l10n.t('Asked {0} questions', questionCount);

	return {
		invocationMessage: new MarkdownString(message),
		pastTenseMessage: new MarkdownString(pastMessage)
	};
}
```

### "Other..." Option Injection

An automatic free-text option is appended to every question:

```typescript
items.push({
	label: vscode.l10n.t('Other...'),
	description: vscode.l10n.t('Enter custom answer'),
	isFreeText: true,
	originalLabel: 'Other'
});
```

When selected, `vscode.window.showInputBox()` captures custom text:

```typescript
const freeTextInput = await vscode.window.showInputBox({
	prompt: question.question,
	placeHolder: vscode.l10n.t('Enter your answer'),
	ignoreFocusOut: true
}, token);
```

**Free-text cancellation behaviour:**
- If free-text input is cancelled but other selections exist, those selections are preserved
- If free-text input is cancelled with no other selections, the question is marked `skipped`
- If free-text input is provided, `selected` array contains other selections (if any) or `['Other']` if none

```typescript
if (freeTextInput === undefined) {
	// User cancelled the input box
	if (otherSelections.length > 0) {
		resolve({ selected: otherSelections, freeText: null, skipped: false });
	} else {
		resolve('skipped');
	}
} else {
	resolve({
		selected: otherSelections.length > 0 ? otherSelections : [freeTextItem.originalLabel],
		freeText: freeTextInput,
		skipped: false
	});
}
```

### Recommended Option Rendering

Options marked `recommended: true` display a star icon:

```typescript
label: opt.recommended ? `$(star-full) ${opt.label}` : opt.label
```

### Multi-Step Navigation

- **Back button:** Displayed when `step > 0`
- **ESC handling:** Marks current and remaining questions as `skipped: true`
- **Sequential flow:** Questions presented one at a time

### Race Condition Protection

A `safeResolve` wrapper prevents double-resolution if `onDidHide` fires after `onDidAccept`:

```typescript
let resolved = false;
const safeResolve = (value: T) => {
	if (!resolved) {
		resolved = true;
		resolve(value);
	}
};
```

### Default Selection Behaviour

When the QuickPick opens, the default/recommended option is pre-selected differently based on mode:

```typescript
if (defaultItem) {
	if (question.multiSelect) {
		quickPick.selectedItems = [defaultItem];  // Pre-checked for multi-select
	} else {
		quickPick.activeItems = [defaultItem];    // Pre-highlighted for single-select
	}
}
```

### Empty Selection Handling

If the user presses Enter with no selection, the default option is returned automatically:

```typescript
if (selectedItems.length === 0) {
	quickPick.hide();
	safeResolve({
		selected: [defaultOption.label],
		freeText: null,
		skipped: false
	});
	return;
}
```

### Original Label Preservation

The `IQuickPickOptionItem` interface includes an `originalLabel` field to preserve the unmodified label (without star prefix) for output:

```typescript
interface IQuickPickOptionItem extends vscode.QuickPickItem {
	originalLabel: string;    // Unmodified label for results
	isRecommended?: boolean;
	isFreeText: boolean;
}
```

### Token Cancellation Check

Before showing the UI, cancellation is checked:

```typescript
if (token.isCancellationRequested) {
	return 'skipped';
}
```

### Disposable Management

The implementation uses `DisposableStore` for proper cleanup of QuickPick and event listeners:

```typescript
const disposables = new DisposableStore();
try {
	// ... QuickPick logic
	disposables.add(quickPick.onDidAccept(...));
	disposables.add(quickPick.onDidHide(...));
	disposables.add(quickPick.onDidTriggerButton(...));
} finally {
	disposables.dispose();
}
```

## Configuration

### Feature Flag

| Setting | Default | Description |
|---------|---------|-------------|
| `github.copilot.chat.askQuestions.enabled` | `true` | Allow agent mode to ask clarifying questions |

### package.json Configuration

```json
"github.copilot.chat.askQuestions.enabled": {
	"type": "boolean",
	"default": true,
	"markdownDescription": "Allow agent mode to ask clarifying questions before proceeding with a task.",
	"tags": ["experimental", "onExp"]
}
```

### Experimentation Tags

| Tag | Purpose |
|-----|---------|
| `experimental` | Marks setting as experimental in Settings UI |
| `onExp` | Enables VS Code experimentation service override |

### Gating Mechanism

Tool visibility is controlled declaratively via the `when` clause:

```json
"when": "config.github.copilot.chat.askQuestions.enabled"
```

## Telemetry

The tool emits `askQuestionsToolInvoked` events with the following metrics:

**String Properties:**

| Property | Description |
|----------|-------------|
| `requestId` | Unique identifier for the tool invocation request |

**Numeric Metrics:**

| Metric | Description |
|--------|-------------|
| `questionCount` | Total questions presented |
| `answeredCount` | Questions answered (not skipped) |
| `skippedCount` | Questions skipped via ESC |
| `freeTextCount` | Questions with free-text responses |
| `recommendedAvailableCount` | Questions with a recommended option |
| `recommendedSelectedCount` | Questions where recommended was selected |
| `duration` | Total time to complete all questions (ms) |

## Error Handling

| Condition | Behavior |
|-----------|----------|
| Empty questions array | Throws error in `prepareInvocation` |
| Question with < 2 options | Throws error in `prepareInvocation` |
| User cancellation (ESC) | Marks remaining questions as `skipped: true` |
| Token cancellation | Aborts and marks remaining as skipped |
| InputBox dismissal | Falls back to non-free-text selections if any |

## File References

| File | Purpose |
|------|---------|
| [src/extension/tools/vscode-node/askQuestionsTool.ts](src/extension/tools/vscode-node/askQuestionsTool.ts) | Main implementation |
| [src/extension/tools/common/toolNames.ts](src/extension/tools/common/toolNames.ts) | Tool name enums |
| [src/extension/tools/common/toolsRegistry.ts](src/extension/tools/common/toolsRegistry.ts) | Registration infrastructure |
| [src/platform/configuration/common/configurationService.ts](src/platform/configuration/common/configurationService.ts) | Configuration definition |
| [package.json](package.json#L1218-L1291) | Tool schema and configuration |

## Usage Guidelines for Model

### When to Use
- Clarify ambiguous requirements before proceeding
- Get user preferences on implementation choices
- Confirm high-impact decisions

### Best Practices
- Batch related questions into a single call (max 4)
- Provide brief context explaining what is being decided
- Mark one option as `recommended` with a short justification
- Keep options mutually exclusive for single-select
- Use `multiSelect: true` only when choices are additive

### After Receiving Answers
- Restate the chosen decisions briefly and proceed
- Do not re-ask unless requirements change

### Automatic Behavior
- An "Other..." option is automatically shown to users for custom input
- Do not add your own "Something else" or similar option

---

## Standalone Extension Portability

### Summary

The `askQuestions` tool is **fully portable** to a standalone VS Code extension. The complete implementation above uses only stable VS Code APIs. Simply copy the code, adjust the namespace/imports, and register in your extension's `activate()` function.

### VS Code API Dependencies

All APIs are **stable** (not proposed):

| API | Namespace | Stability |
|-----|-----------|-----------|
| `window.createQuickPick<T>()` | `window` | Stable |
| `window.showInputBox()` | `window` | Stable |
| `QuickPickItem` | `window` | Stable |
| `QuickInputButtons.Back` | `window` | Stable |
| `l10n.t()` | `l10n` | Stable |
| `LanguageModelTool<T>` | `lm` | Stable (VS Code 1.93+) |
| `LanguageModelToolInvocationOptions<T>` | `lm` | Stable |
| `LanguageModelToolResult` | `lm` | Stable |
| `PreparedToolInvocation` | `lm` | Stable |
| `MarkdownString` | Core | Stable |
| `CancellationToken` | Core | Stable |

### Blocking Issues

**None.** All APIs are stable, dependencies are optional or portable.

### Minimum VS Code Version

**1.93** (when `LanguageModelTool` API was stabilised)
