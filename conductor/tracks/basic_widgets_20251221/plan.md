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
- [ ] Task: Design Static base widget with markup support
- [ ] Task: Design reactive property integration for all widgets
- [ ] Task: Design DEFAULT_CSS pattern in Rust
- [ ] Task: Design message types matching Python Textual exactly
- [ ] Task: Design Validator and Suggester traits for Input
- [ ] Task: Write technical design document
- [ ] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md)

## Phase 3: Implementation

### 3.1 Static Widget (Foundation for Label)
- [ ] Task: Implement Static struct with content property
- [ ] Task: Implement markup parsing support
- [ ] Task: Implement expand/shrink properties
- [ ] Task: Implement update() method
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Static

### 3.2 Label Widget (REBUILD - extends Static)
- [ ] Task: Remove existing Label implementation
- [ ] Task: Implement Label extending Static
- [ ] Task: Implement LabelVariant support
- [ ] Task: Add variant class styling
- [ ] Task: Write tests for Label

### 3.3 Button Widget (REBUILD)
- [ ] Task: Remove existing Button implementation
- [ ] Task: Implement Button with ButtonVariant
- [ ] Task: Implement label reactive property
- [ ] Task: Implement action parameter support
- [ ] Task: Implement disabled state
- [ ] Task: Add DEFAULT_CSS with all variant styles
- [ ] Task: Implement Pressed message with button reference
- [ ] Task: Write tests for Button

### 3.4 Input Widget (REBUILD - extends ScrollView)
- [ ] Task: Remove existing Input implementation
- [ ] Task: Implement Input extending ScrollView
- [ ] Task: Implement value, placeholder, password properties
- [ ] Task: Implement Selection with start/end
- [ ] Task: Implement restrict regex pattern
- [ ] Task: Implement InputType (integer, number, text)
- [ ] Task: Implement max_length
- [ ] Task: Implement Validator trait and validation
- [ ] Task: Implement Suggester trait and autocomplete
- [ ] Task: Implement all cursor movement bindings
- [ ] Task: Implement Changed message with validation_result
- [ ] Task: Implement Submitted message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Input

### 3.5 Header Widget (REBUILD)
- [ ] Task: Remove existing Header implementation
- [ ] Task: Implement Header with screen_title integration
- [ ] Task: Implement screen_sub_title
- [ ] Task: Implement show_clock and time_format
- [ ] Task: Implement icon display
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Header

### 3.6 Footer Widget (REBUILD)
- [ ] Task: Remove existing Footer implementation
- [ ] Task: Implement Footer with binding collection
- [ ] Task: Implement compact mode
- [ ] Task: Implement show_command_palette
- [ ] Task: Implement binding priority filtering
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Footer

### 3.7 Checkbox Widget
- [ ] Task: Implement Checkbox with value reactive property
- [ ] Task: Implement label property
- [ ] Task: Implement visual rendering (☐/☑)
- [ ] Task: Implement Changed message
- [ ] Task: Add Space key binding for toggle
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Checkbox

### 3.8 RadioButton Widget
- [ ] Task: Implement RadioButton with value property
- [ ] Task: Implement label property
- [ ] Task: Implement visual rendering (○/●)
- [ ] Task: Write tests for RadioButton

### 3.9 RadioSet Widget
- [ ] Task: Implement RadioSet container
- [ ] Task: Implement exclusive selection logic
- [ ] Task: Implement arrow key navigation
- [ ] Task: Implement Changed message with pressed button
- [ ] Task: Write tests for RadioSet

### 3.10 Switch Widget
- [ ] Task: Implement Switch with value property
- [ ] Task: Implement animate property
- [ ] Task: Implement sliding animation
- [ ] Task: Implement Changed message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Switch

### 3.11 Select Widget
- [ ] Task: Implement Select with options list
- [ ] Task: Implement value reactive property
- [ ] Task: Implement prompt property
- [ ] Task: Implement allow_blank
- [ ] Task: Implement dropdown behavior
- [ ] Task: Implement Changed message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Select

### 3.12 OptionList Widget
- [ ] Task: Implement OptionList with options
- [ ] Task: Implement highlighted property
- [ ] Task: Implement keyboard navigation
- [ ] Task: Implement OptionHighlighted message
- [ ] Task: Implement OptionSelected message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for OptionList

### 3.13 SelectionList Widget
- [ ] Task: Implement SelectionList for multi-select
- [ ] Task: Implement selected list
- [ ] Task: Implement SelectedChanged message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for SelectionList

### 3.14 ProgressBar Widget
- [ ] Task: Implement ProgressBar with progress property
- [ ] Task: Implement total property
- [ ] Task: Implement show_bar, show_percentage, show_eta
- [ ] Task: Implement advance() method
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for ProgressBar

### 3.15 LoadingIndicator Widget
- [ ] Task: Implement LoadingIndicator
- [ ] Task: Implement animation frames
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for LoadingIndicator

### 3.16 Rule Widget
- [ ] Task: Implement Rule with orientation
- [ ] Task: Implement line_style options
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Rule

### 3.17 Link Widget
- [ ] Task: Implement Link with text and url
- [ ] Task: Implement Clicked message
- [ ] Task: Implement URL opening
- [ ] Task: Add hover styling
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Link

### 3.18 Utility Widgets
- [ ] Task: Implement Placeholder widget
- [ ] Task: Implement Digits widget
- [ ] Task: Implement Sparkline widget
- [ ] Task: Implement Pretty widget
- [ ] Task: Write tests for utility widgets

- [ ] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Testing & Verification
- [ ] Task: Create API compatibility tests
- [ ] Task: Create example app with all widgets
- [ ] Task: Test keyboard navigation
- [ ] Task: Run all tests and fix failures
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
