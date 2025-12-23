# Plan: Basic Widgets

## Phase 1: Analysis & Research
- [x] Task: Study Python Textual _static.py as base for Label [cc15181]
- [x] Task: Analyze _button.py for variants, action support, DEFAULT_CSS [2061a96]
- [x] Task: Study _input.py for ScrollView inheritance, selection, validation, suggester [0b89586]
- [x] Task: Document _header.py clock, icon, screen title integration [f09dba5]
- [x] Task: Analyze _footer.py binding collection and display [f09dba5]
- [x] Task: Study _checkbox.py, _radio_button.py, _switch.py patterns [32adbeb]
- [x] Task: Analyze _select.py, _option_list.py, _selection_list.py [32adbeb]
- [x] Task: Document _progress_bar.py, _loading_indicator.py [32adbeb]
- [x] Task: Study _rule.py, _link.py [32adbeb]
- [x] Task: Address code review findings (5 issues fixed) [c448326]
- [x] Task: Address code review round 2 (6 issues fixed) [bb36ede]
- [x] Task: Address code review round 3 (6 issues fixed) [9589270]
- [x] Task: Address code review round 4 (3 issues fixed) [6df6be2]
- [x] Task: Address code review round 5 (4 issues fixed) [212bd43]
- [x] Task: Address code review round 6 (3 issues fixed) [aed37a6]
- [x] Task: Conductor - User Manual Verification 'Analysis Complete' (Approved with residual risk note)

## Phase 2: Design & Planning
- [x] Task: Design Static base widget with markup support [715bdde]
- [x] Task: Design reactive property integration for all widgets [00de357]
- [x] Task: Design DEFAULT_CSS pattern in Rust [6820188]
- [x] Task: Design message types matching Python Textual exactly [3b88b76]
- [x] Task: Design Validator and Suggester traits for Input [2a796c2]
- [x] Task: Write technical design document [7d69c43]
- [x] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md) [93121e2]

## Phase 3: Implementation [checkpoint: 25a0187]

### 3.1 Static Widget (Foundation for Label)
- [x] Task: Implement Static struct with content property [0d51b8c]
- [x] Task: Implement markup parsing support [0d51b8c]
- [x] Task: Implement expand/shrink properties [0d51b8c]
- [x] Task: Implement update() method [0d51b8c]
- [x] Task: Add DEFAULT_CSS [0d51b8c]
- [x] Task: Write tests for Static [0d51b8c]

### 3.2 Label Widget (REBUILD - extends Static)
- [x] Task: Remove existing Label implementation [1976105]
- [x] Task: Implement Label extending Static [1976105]
- [x] Task: Implement LabelVariant support [1976105]
- [x] Task: Add variant class styling [1976105]
- [x] Task: Write tests for Label [1976105]

### 3.3 Button Widget (REBUILD)
- [x] Task: Remove existing Button implementation [bcb97f9]
- [x] Task: Implement Button with ButtonVariant [bcb97f9]
- [x] Task: Implement label reactive property [bcb97f9]
- [x] Task: Implement action parameter support [bcb97f9]
- [x] Task: Implement disabled state [bcb97f9]
- [x] Task: Add DEFAULT_CSS with all variant styles [bcb97f9]
- [x] Task: Implement Pressed message with button reference [bcb97f9]
- [x] Task: Write tests for Button [bcb97f9]

### 3.4 Input Widget (REBUILD - extends ScrollView)
- [x] Task: Remove existing Input implementation [cf42a57]
- [x] Task: Implement Input extending ScrollView [cf42a57]
- [x] Task: Implement value, placeholder, password properties [cf42a57]
- [x] Task: Implement Selection with start/end [cf42a57]
- [x] Task: Implement restrict regex pattern [cf42a57]
- [x] Task: Implement InputType (integer, number, text) [cf42a57]
- [x] Task: Implement max_length [cf42a57]
- [x] Task: Implement Validator trait and validation [cf42a57]
- [ ] Task: Implement Suggester trait and autocomplete (deferred)
- [x] Task: Implement all cursor movement bindings [cf42a57]
- [x] Task: Implement Changed message with validation_result [cf42a57]
- [x] Task: Implement Submitted message [cf42a57]
- [x] Task: Add DEFAULT_CSS [cf42a57]
- [x] Task: Write tests for Input [cf42a57]

### 3.5 Header Widget (REBUILD)
- [x] Task: Remove existing Header implementation [7e4966c]
- [x] Task: Implement Header with screen_title integration [7e4966c]
- [x] Task: Implement screen_sub_title [7e4966c]
- [x] Task: Implement show_clock and time_format [7e4966c]
- [x] Task: Implement icon display [7e4966c]
- [x] Task: Add DEFAULT_CSS [7e4966c]
- [x] Task: Write tests for Header [7e4966c]

### 3.6 Footer Widget (REBUILD)
- [x] Task: Remove existing Footer implementation [eaf3c1f]
- [x] Task: Implement Footer with binding collection [eaf3c1f]
- [x] Task: Implement compact mode [eaf3c1f]
- [x] Task: Implement show_command_palette [eaf3c1f]
- [x] Task: Implement binding priority filtering [eaf3c1f]
- [x] Task: Add DEFAULT_CSS [eaf3c1f]
- [x] Task: Write tests for Footer [eaf3c1f]

### 3.7 Checkbox Widget
- [x] Task: Implement Checkbox with value reactive property [1c94376]
- [x] Task: Implement label property [1c94376]
- [x] Task: Implement visual rendering (▐X▌/▐ ▌) [1c94376]
- [x] Task: Implement Changed message [1c94376]
- [x] Task: Add Space key binding for toggle [1c94376]
- [x] Task: Add DEFAULT_CSS [1c94376]
- [x] Task: Write tests for Checkbox [1c94376]

### 3.8 RadioButton Widget
- [x] Task: Implement RadioButton with value property [63fde8f]
- [x] Task: Implement label property [63fde8f]
- [x] Task: Implement visual rendering (▐●▌/▐ ▌) [63fde8f]
- [x] Task: Write tests for RadioButton [63fde8f]

### 3.9 RadioSet Widget
- [x] Task: Implement RadioSet container [7de6659]
- [x] Task: Implement exclusive selection logic [7de6659]
- [x] Task: Implement arrow key navigation [7de6659]
- [x] Task: Implement Changed message with pressed button [7de6659]
- [x] Task: Write tests for RadioSet [7de6659]

### 3.10 Switch Widget
- [x] Task: Implement Switch with value property [e40469a]
- [x] Task: Implement animate property [e40469a]
- [x] Task: Implement sliding animation (deferred to timer system) [e40469a]
- [x] Task: Implement Changed message [e40469a]
- [x] Task: Add DEFAULT_CSS [e40469a]
- [x] Task: Write tests for Switch [e40469a]

### 3.11 Select Widget
- [x] Task: Implement Select with options list [8cc474e]
- [x] Task: Implement value reactive property [8cc474e]
- [x] Task: Implement prompt property [8cc474e]
- [x] Task: Implement allow_blank [8cc474e]
- [x] Task: Implement dropdown behavior [8cc474e]
- [x] Task: Implement Changed message [8cc474e]
- [x] Task: Add DEFAULT_CSS [8cc474e]
- [x] Task: Write tests for Select [8cc474e]

### 3.12 OptionList Widget
- [x] Task: Implement OptionList with options [6aa655e]
- [x] Task: Implement highlighted property [6aa655e]
- [x] Task: Implement keyboard navigation [6aa655e]
- [x] Task: Implement OptionHighlighted message [6aa655e]
- [x] Task: Implement OptionSelected message [6aa655e]
- [x] Task: Add DEFAULT_CSS [6aa655e]
- [x] Task: Write tests for OptionList (31 tests) [6aa655e]

### 3.13 SelectionList Widget
- [x] Task: Implement SelectionList for multi-select [acad7d6]
- [x] Task: Implement selected list [acad7d6]
- [x] Task: Implement SelectedChanged message [acad7d6]
- [x] Task: Add DEFAULT_CSS [acad7d6]
- [x] Task: Write tests for SelectionList (28 tests) [acad7d6]

### 3.14 ProgressBar Widget
- [x] Task: Implement ProgressBar with progress property [6ec7d23]
- [x] Task: Implement total property [6ec7d23]
- [x] Task: Implement show_bar, show_percentage, show_eta [6ec7d23]
- [x] Task: Implement advance() method [6ec7d23]
- [x] Task: Add DEFAULT_CSS [6ec7d23]
- [x] Task: Write tests for ProgressBar [6ec7d23]

### 3.15 Timer System Infrastructure
- [x] Task: Review and approve timer system design (design/timer_system.md)
- [x] Task: Implement TimerId struct with atomic counter [ef09cd7]
- [x] Task: Implement TimerTick message in pump::events [ef09cd7]
- [x] Task: Implement Timer struct with start/stop/pause/resume [ef09cd7]
- [x] Task: Implement Drop for Timer cleanup [ef09cd7]
- [x] Task: Add set_timer() method to MessagePump [ef09cd7]
- [x] Task: Add set_interval() method to MessagePump [ef09cd7]
- [x] Task: Write unit tests for Timer (36 tests) [ef09cd7]
- [x] Task: Write integration tests for timer behavior (5 tests) [ef09cd7]
- [x] Task: Address code review findings (timer/pump integration, handler_name, flaky tests) [12f5490]

### 3.16 LoadingIndicator Widget
- [x] Task: Implement LoadingIndicator [3d6dfee]
- [x] Task: Implement animation frames using timer system [3d6dfee]
- [x] Task: Add DEFAULT_CSS [3d6dfee]
- [x] Task: Write tests for LoadingIndicator (22 tests) [3d6dfee]

### 3.17 Rule Widget
- [x] Task: Implement Rule with orientation [c48abf8]
- [x] Task: Implement line_style options [c48abf8]
- [x] Task: Add DEFAULT_CSS [c48abf8]
- [x] Task: Write tests for Rule [c48abf8]

### 3.18 Link Widget
- [x] Task: Implement Link with text and url [0c98909]
- [x] Task: Implement Clicked message [0c98909]
- [x] Task: Implement URL opening (gated behind open-url feature) [0c98909]
- [x] Task: Add hover styling [0c98909]
- [x] Task: Add DEFAULT_CSS [0c98909]
- [x] Task: Write tests for Link (24 tests) [0c98909]

### 3.19 Utility Widgets
- [x] Task: Implement Placeholder widget (18 tests) [ec35388]
- [x] Task: Implement Digits widget (18 tests) [ec35388]
- [x] Task: Implement Sparkline widget (26 tests) [ec35388]
- [x] Task: Implement Pretty widget (14 tests) [ec35388]
- [x] Task: Write tests for utility widgets (76 tests total) [ec35388]

- [x] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md) [25a0187]

## Phase 4: Testing & Verification
- [ ] Task: Create API compatibility tests
- [ ] Task: Create example app with all widgets
- [ ] Task: Test keyboard navigation
- [ ] Task: Run all tests and fix failures
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
