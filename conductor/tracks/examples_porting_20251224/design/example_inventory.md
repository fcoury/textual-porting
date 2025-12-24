# Python Example Inventory

This inventory captures all Python Textual examples and assets that need mapping to Rust ports.

## 1) `textual/examples/` (Core Examples)

| Example | Assets | Notes |
| --- | --- | --- |
| calculator.py | calculator.tcss | Core parity target (already ported) |
| breakpoints.py | — | Layout + resize behavior |
| clock.py | — | Reactive clock display |
| code_browser.py | code_browser.tcss | Tree/list + styling |
| color_command.py | — | Command palette / color UI |
| dictionary.py | dictionary.tcss, food.json | Data-driven list |
| five_by_five.py | five_by_five.tcss, five_by_five.md | Grid + markdown |
| json_tree.py | food.json | Tree example |
| markdown.py | example.md | Markdown rendering |
| merlin.py | — | Showcase / animations |
| mother.py | — | Showcase / layout |
| pride.py | — | Theme colors / gradients |
| sidebar.py | — | Layout / dock |
| splash.py | — | Intro UI |
| theme_sandbox.py | — | Theme tuning |

Supporting docs and data:
- README.md
- demo.md
- example.md
- five_by_five.md
- food.json

## 2) `textual/src/textual/demo/` (Demo App)

| Module | Notes |
| --- | --- |
| __main__.py | Demo entrypoint |
| demo_app.py | App shell + routing |
| home.py | Home page |
| projects.py | Projects page |
| page.py | Generic page base |
| widgets.py | Widgets gallery page |
| game.py | Demo game |
| data.py | Demo data access |
| _project_data.py | Project data backing |
| _project_stars.py | GitHub star cache |
| _project_stargazer_updater.py | Data update helper |

## 3) `textual/docs/examples/` (Docs Examples)

Docs examples are numerous and grouped by topic. The full list below should be used
for mapping to Rust ports, skipping duplicates where an example is already covered
by a core example or demo page.

Full inventory (generated from repo):
```
textual/docs/examples/app/event01.py
textual/docs/examples/app/question01.py
textual/docs/examples/app/question02.py
textual/docs/examples/app/question02.tcss
textual/docs/examples/app/question03.py
textual/docs/examples/app/question_title01.py
textual/docs/examples/app/question_title02.py
textual/docs/examples/app/simple01.py
textual/docs/examples/app/simple02.py
textual/docs/examples/app/suspend.py
textual/docs/examples/app/suspend_process.py
textual/docs/examples/app/widgets01.py
textual/docs/examples/app/widgets02.py
textual/docs/examples/app/widgets03.py
textual/docs/examples/app/widgets04.py
textual/docs/examples/events/custom01.py
textual/docs/examples/events/dictionary.py
textual/docs/examples/events/dictionary.tcss
textual/docs/examples/events/on_decorator.tcss
textual/docs/examples/events/on_decorator01.py
textual/docs/examples/events/on_decorator02.py
textual/docs/examples/events/prevent.py
textual/docs/examples/getting_started/console.py
textual/docs/examples/guide/actions/actions01.py
textual/docs/examples/guide/actions/actions02.py
textual/docs/examples/guide/actions/actions03.py
textual/docs/examples/guide/actions/actions04.py
textual/docs/examples/guide/actions/actions05.py
textual/docs/examples/guide/actions/actions05.tcss
textual/docs/examples/guide/actions/actions06.py
textual/docs/examples/guide/actions/actions06.tcss
textual/docs/examples/guide/actions/actions07.py
textual/docs/examples/guide/animator/animation01.py
textual/docs/examples/guide/animator/animation01_static.py
textual/docs/examples/guide/command_palette/command01.py
textual/docs/examples/guide/command_palette/command02.py
textual/docs/examples/guide/compound/byte01.py
textual/docs/examples/guide/compound/byte02.py
textual/docs/examples/guide/compound/byte03.py
textual/docs/examples/guide/compound/compound01.py
textual/docs/examples/guide/content/content01.py
textual/docs/examples/guide/content/playground.py
textual/docs/examples/guide/content/renderables.py
textual/docs/examples/guide/css/nesting01.py
textual/docs/examples/guide/css/nesting01.tcss
textual/docs/examples/guide/css/nesting02.py
textual/docs/examples/guide/css/nesting02.tcss
textual/docs/examples/guide/dom1.py
textual/docs/examples/guide/dom2.py
textual/docs/examples/guide/dom3.py
textual/docs/examples/guide/dom4.py
textual/docs/examples/guide/dom4.tcss
textual/docs/examples/guide/input/binding01.py
textual/docs/examples/guide/input/binding01.tcss
textual/docs/examples/guide/input/key01.py
textual/docs/examples/guide/input/key02.py
textual/docs/examples/guide/input/key03.py
textual/docs/examples/guide/input/key03.tcss
textual/docs/examples/guide/input/mouse01.py
textual/docs/examples/guide/input/mouse01.tcss
textual/docs/examples/guide/layout/combining_layouts.py
textual/docs/examples/guide/layout/combining_layouts.tcss
textual/docs/examples/guide/layout/dock_layout1_sidebar.py
textual/docs/examples/guide/layout/dock_layout1_sidebar.tcss
textual/docs/examples/guide/layout/dock_layout2_sidebar.py
textual/docs/examples/guide/layout/dock_layout2_sidebar.tcss
textual/docs/examples/guide/layout/dock_layout3_sidebar_header.py
textual/docs/examples/guide/layout/dock_layout3_sidebar_header.tcss
textual/docs/examples/guide/layout/grid_layout1.py
textual/docs/examples/guide/layout/grid_layout1.tcss
textual/docs/examples/guide/layout/grid_layout2.py
textual/docs/examples/guide/layout/grid_layout2.tcss
textual/docs/examples/guide/layout/grid_layout3_row_col_adjust.py
textual/docs/examples/guide/layout/grid_layout3_row_col_adjust.tcss
textual/docs/examples/guide/layout/grid_layout4_row_col_adjust.py
textual/docs/examples/guide/layout/grid_layout4_row_col_adjust.tcss
textual/docs/examples/guide/layout/grid_layout5_col_span.py
textual/docs/examples/guide/layout/grid_layout5_col_span.tcss
textual/docs/examples/guide/layout/grid_layout6_row_span.py
textual/docs/examples/guide/layout/grid_layout6_row_span.tcss
textual/docs/examples/guide/layout/grid_layout7_gutter.py
textual/docs/examples/guide/layout/grid_layout7_gutter.tcss
textual/docs/examples/guide/layout/grid_layout_auto.py
textual/docs/examples/guide/layout/grid_layout_auto.tcss
textual/docs/examples/guide/layout/horizontal_layout.py
textual/docs/examples/guide/layout/horizontal_layout.tcss
textual/docs/examples/guide/layout/horizontal_layout_overflow.py
textual/docs/examples/guide/layout/horizontal_layout_overflow.tcss
textual/docs/examples/guide/layout/layers.py
textual/docs/examples/guide/layout/layers.tcss
textual/docs/examples/guide/layout/utility_containers.py
textual/docs/examples/guide/layout/utility_containers.tcss
textual/docs/examples/guide/layout/utility_containers_using_with.py
textual/docs/examples/guide/layout/vertical_layout.py
textual/docs/examples/guide/layout/vertical_layout.tcss
textual/docs/examples/guide/layout/vertical_layout_scrolled.py
textual/docs/examples/guide/layout/vertical_layout_scrolled.tcss
textual/docs/examples/guide/reactivity/computed01.py
textual/docs/examples/guide/reactivity/computed01.tcss
textual/docs/examples/guide/reactivity/dynamic_watch.py
textual/docs/examples/guide/reactivity/recompose01.py
textual/docs/examples/guide/reactivity/recompose02.py
textual/docs/examples/guide/reactivity/refresh01.py
textual/docs/examples/guide/reactivity/refresh01.tcss
textual/docs/examples/guide/reactivity/refresh02.py
textual/docs/examples/guide/reactivity/refresh02.tcss
textual/docs/examples/guide/reactivity/refresh03.py
textual/docs/examples/guide/reactivity/refresh03.tcss
textual/docs/examples/guide/reactivity/set_reactive01.py
textual/docs/examples/guide/reactivity/set_reactive02.py
textual/docs/examples/guide/reactivity/set_reactive03.py
textual/docs/examples/guide/reactivity/validate01.py
textual/docs/examples/guide/reactivity/validate01.tcss
textual/docs/examples/guide/reactivity/watch01.py
textual/docs/examples/guide/reactivity/watch01.tcss
textual/docs/examples/guide/reactivity/world_clock01.py
textual/docs/examples/guide/reactivity/world_clock01.tcss
textual/docs/examples/guide/reactivity/world_clock02.py
textual/docs/examples/guide/reactivity/world_clock03.py
textual/docs/examples/guide/screens/modal01.py
textual/docs/examples/guide/screens/modal01.tcss
textual/docs/examples/guide/screens/modal02.py
textual/docs/examples/guide/screens/modal03.py
textual/docs/examples/guide/screens/modes01.py
textual/docs/examples/guide/screens/questions01.py
textual/docs/examples/guide/screens/questions01.tcss
textual/docs/examples/guide/screens/screen01.py
textual/docs/examples/guide/screens/screen01.tcss
textual/docs/examples/guide/screens/screen02.py
textual/docs/examples/guide/screens/screen02.tcss
textual/docs/examples/guide/structure.py
textual/docs/examples/guide/styles/border01.py
textual/docs/examples/guide/styles/border_title.py
textual/docs/examples/guide/styles/box_sizing01.py
textual/docs/examples/guide/styles/colors.py
textual/docs/examples/guide/styles/colors01.py
textual/docs/examples/guide/styles/colors02.py
textual/docs/examples/guide/styles/dimensions01.py
textual/docs/examples/guide/styles/dimensions02.py
textual/docs/examples/guide/styles/dimensions03.py
textual/docs/examples/guide/styles/dimensions04.py
textual/docs/examples/guide/styles/margin01.py
textual/docs/examples/guide/styles/outline01.py
textual/docs/examples/guide/styles/padding01.py
textual/docs/examples/guide/styles/padding02.py
textual/docs/examples/guide/styles/screen.py
textual/docs/examples/guide/styles/widget.py
textual/docs/examples/guide/testing/rgb.py
textual/docs/examples/guide/testing/test_rgb.py
textual/docs/examples/guide/widgets/checker01.py
textual/docs/examples/guide/widgets/checker02.py
textual/docs/examples/guide/widgets/checker03.py
textual/docs/examples/guide/widgets/checker04.py
textual/docs/examples/guide/widgets/counter.tcss
textual/docs/examples/guide/widgets/counter01.py
textual/docs/examples/guide/widgets/counter02.py
textual/docs/examples/guide/widgets/fizzbuzz01.py
textual/docs/examples/guide/widgets/fizzbuzz01.tcss
textual/docs/examples/guide/widgets/fizzbuzz02.py
textual/docs/examples/guide/widgets/fizzbuzz02.tcss
textual/docs/examples/guide/widgets/hello01.py
textual/docs/examples/guide/widgets/hello01.tcss
textual/docs/examples/guide/widgets/hello02.py
textual/docs/examples/guide/widgets/hello02.tcss
textual/docs/examples/guide/widgets/hello03.py
textual/docs/examples/guide/widgets/hello03.tcss
textual/docs/examples/guide/widgets/hello04.py
textual/docs/examples/guide/widgets/hello04.tcss
textual/docs/examples/guide/widgets/hello05.py
textual/docs/examples/guide/widgets/hello05.tcss
textual/docs/examples/guide/widgets/hello06.py
textual/docs/examples/guide/widgets/hello06.tcss
textual/docs/examples/guide/widgets/loading01.py
textual/docs/examples/guide/widgets/tooltip01.py
textual/docs/examples/guide/widgets/tooltip02.py
textual/docs/examples/guide/workers/weather.tcss
textual/docs/examples/guide/workers/weather01.py
textual/docs/examples/guide/workers/weather02.py
textual/docs/examples/guide/workers/weather03.py
textual/docs/examples/guide/workers/weather04.py
textual/docs/examples/guide/workers/weather05.py
textual/docs/examples/how-to/center01.py
textual/docs/examples/how-to/center02.py
textual/docs/examples/how-to/center03.py
textual/docs/examples/how-to/center04.py
textual/docs/examples/how-to/center05.py
textual/docs/examples/how-to/center06.py
textual/docs/examples/how-to/center07.py
textual/docs/examples/how-to/center08.py
textual/docs/examples/how-to/center09.py
textual/docs/examples/how-to/center10.py
textual/docs/examples/how-to/containers01.py
textual/docs/examples/how-to/containers02.py
textual/docs/examples/how-to/containers03.py
textual/docs/examples/how-to/containers04.py
textual/docs/examples/how-to/containers05.py
textual/docs/examples/how-to/containers06.py
textual/docs/examples/how-to/containers07.py
textual/docs/examples/how-to/containers08.py
textual/docs/examples/how-to/containers09.py
textual/docs/examples/how-to/inline01.py
textual/docs/examples/how-to/inline02.py
textual/docs/examples/how-to/layout.py
textual/docs/examples/how-to/layout01.py
textual/docs/examples/how-to/layout02.py
textual/docs/examples/how-to/layout03.py
textual/docs/examples/how-to/layout04.py
textual/docs/examples/how-to/layout05.py
textual/docs/examples/how-to/layout06.py
textual/docs/examples/how-to/render_compose.py
textual/docs/examples/styles/README.md
textual/docs/examples/styles/align.py
textual/docs/examples/styles/align.tcss
textual/docs/examples/styles/align_all.py
textual/docs/examples/styles/align_all.tcss
textual/docs/examples/styles/background.py
textual/docs/examples/styles/background.tcss
textual/docs/examples/styles/background_tint.py
textual/docs/examples/styles/background_tint.tcss
textual/docs/examples/styles/background_transparency.py
textual/docs/examples/styles/background_transparency.tcss
textual/docs/examples/styles/border.py
textual/docs/examples/styles/border.tcss
textual/docs/examples/styles/border_all.py
textual/docs/examples/styles/border_all.tcss
textual/docs/examples/styles/border_sub_title_align_all.py
textual/docs/examples/styles/border_sub_title_align_all.tcss
textual/docs/examples/styles/border_subtitle_align.py
textual/docs/examples/styles/border_subtitle_align.tcss
textual/docs/examples/styles/border_title_align.py
textual/docs/examples/styles/border_title_align.tcss
textual/docs/examples/styles/border_title_colors.py
textual/docs/examples/styles/border_title_colors.tcss
textual/docs/examples/styles/box_sizing.py
textual/docs/examples/styles/box_sizing.tcss
textual/docs/examples/styles/color.py
textual/docs/examples/styles/color.tcss
textual/docs/examples/styles/color_auto.py
textual/docs/examples/styles/color_auto.tcss
textual/docs/examples/styles/column_span.py
textual/docs/examples/styles/column_span.tcss
textual/docs/examples/styles/content_align.py
textual/docs/examples/styles/content_align.tcss
textual/docs/examples/styles/content_align_all.py
textual/docs/examples/styles/content_align_all.tcss
textual/docs/examples/styles/display.py
textual/docs/examples/styles/display.tcss
textual/docs/examples/styles/dock_all.py
textual/docs/examples/styles/dock_all.tcss
textual/docs/examples/styles/grid.py
textual/docs/examples/styles/grid.tcss
textual/docs/examples/styles/grid_columns.py
textual/docs/examples/styles/grid_columns.tcss
textual/docs/examples/styles/grid_gutter.py
textual/docs/examples/styles/grid_gutter.tcss
textual/docs/examples/styles/grid_rows.py
textual/docs/examples/styles/grid_rows.tcss
textual/docs/examples/styles/grid_size_both.py
textual/docs/examples/styles/grid_size_both.tcss
textual/docs/examples/styles/grid_size_columns.py
textual/docs/examples/styles/grid_size_columns.tcss
textual/docs/examples/styles/hatch.py
textual/docs/examples/styles/hatch.tcss
textual/docs/examples/styles/height.py
textual/docs/examples/styles/height.tcss
textual/docs/examples/styles/height_comparison.py
textual/docs/examples/styles/height_comparison.tcss
textual/docs/examples/styles/keyline.py
textual/docs/examples/styles/keyline.tcss
textual/docs/examples/styles/keyline_horizontal.py
textual/docs/examples/styles/keyline_horizontal.tcss
textual/docs/examples/styles/layout.py
textual/docs/examples/styles/layout.tcss
textual/docs/examples/styles/link_background.py
textual/docs/examples/styles/link_background.tcss
textual/docs/examples/styles/link_background_hover.py
textual/docs/examples/styles/link_background_hover.tcss
textual/docs/examples/styles/link_color.py
textual/docs/examples/styles/link_color.tcss
textual/docs/examples/styles/link_color_hover.py
textual/docs/examples/styles/link_color_hover.tcss
textual/docs/examples/styles/link_style.py
textual/docs/examples/styles/link_style.tcss
textual/docs/examples/styles/link_style_hover.py
textual/docs/examples/styles/link_style_hover.tcss
textual/docs/examples/styles/links.py
textual/docs/examples/styles/links.tcss
textual/docs/examples/styles/margin.py
textual/docs/examples/styles/margin.tcss
textual/docs/examples/styles/margin_all.py
textual/docs/examples/styles/margin_all.tcss
textual/docs/examples/styles/max_height.py
textual/docs/examples/styles/max_height.tcss
textual/docs/examples/styles/max_width.py
textual/docs/examples/styles/max_width.tcss
textual/docs/examples/styles/min_height.py
textual/docs/examples/styles/min_height.tcss
textual/docs/examples/styles/min_width.py
textual/docs/examples/styles/min_width.tcss
textual/docs/examples/styles/offset.py
textual/docs/examples/styles/offset.tcss
textual/docs/examples/styles/opacity.py
textual/docs/examples/styles/opacity.tcss
textual/docs/examples/styles/outline.py
textual/docs/examples/styles/outline.tcss
textual/docs/examples/styles/outline_all.py
textual/docs/examples/styles/outline_all.tcss
textual/docs/examples/styles/outline_vs_border.py
textual/docs/examples/styles/outline_vs_border.tcss
textual/docs/examples/styles/overflow.py
textual/docs/examples/styles/overflow.tcss
textual/docs/examples/styles/padding.py
textual/docs/examples/styles/padding.tcss
textual/docs/examples/styles/padding_all.py
textual/docs/examples/styles/padding_all.tcss
textual/docs/examples/styles/position.py
textual/docs/examples/styles/position.tcss
textual/docs/examples/styles/row_span.py
textual/docs/examples/styles/row_span.tcss
textual/docs/examples/styles/scrollbar_corner_color.py
textual/docs/examples/styles/scrollbar_corner_color.tcss
textual/docs/examples/styles/scrollbar_gutter.py
textual/docs/examples/styles/scrollbar_gutter.tcss
textual/docs/examples/styles/scrollbar_size.py
textual/docs/examples/styles/scrollbar_size.tcss
textual/docs/examples/styles/scrollbar_size2.py
textual/docs/examples/styles/scrollbar_size2.tcss
textual/docs/examples/styles/scrollbar_visibility.py
textual/docs/examples/styles/scrollbar_visibility.tcss
textual/docs/examples/styles/scrollbars.py
textual/docs/examples/styles/scrollbars.tcss
textual/docs/examples/styles/scrollbars2.py
textual/docs/examples/styles/scrollbars2.tcss
textual/docs/examples/styles/text_align.py
textual/docs/examples/styles/text_align.tcss
textual/docs/examples/styles/text_opacity.py
textual/docs/examples/styles/text_opacity.tcss
textual/docs/examples/styles/text_overflow.py
textual/docs/examples/styles/text_overflow.tcss
textual/docs/examples/styles/text_style.py
textual/docs/examples/styles/text_style.tcss
textual/docs/examples/styles/text_style_all.py
textual/docs/examples/styles/text_style_all.tcss
textual/docs/examples/styles/text_wrap.py
textual/docs/examples/styles/text_wrap.tcss
textual/docs/examples/styles/tint.py
textual/docs/examples/styles/tint.tcss
textual/docs/examples/styles/visibility.py
textual/docs/examples/styles/visibility.tcss
textual/docs/examples/styles/visibility_containers.py
textual/docs/examples/styles/visibility_containers.tcss
textual/docs/examples/styles/width.py
textual/docs/examples/styles/width.tcss
textual/docs/examples/styles/width_comparison.py
textual/docs/examples/styles/width_comparison.tcss
textual/docs/examples/themes/colored_text.py
textual/docs/examples/themes/muted_backgrounds.py
textual/docs/examples/themes/todo_app.py
textual/docs/examples/tutorial/stopwatch.py
textual/docs/examples/tutorial/stopwatch.tcss
textual/docs/examples/tutorial/stopwatch01.py
textual/docs/examples/tutorial/stopwatch02.py
textual/docs/examples/tutorial/stopwatch02.tcss
textual/docs/examples/tutorial/stopwatch03.py
textual/docs/examples/tutorial/stopwatch03.tcss
textual/docs/examples/tutorial/stopwatch04.py
textual/docs/examples/tutorial/stopwatch04.tcss
textual/docs/examples/tutorial/stopwatch05.py
textual/docs/examples/tutorial/stopwatch06.py
textual/docs/examples/widgets/button.py
textual/docs/examples/widgets/button.tcss
textual/docs/examples/widgets/checkbox.py
textual/docs/examples/widgets/checkbox.tcss
textual/docs/examples/widgets/clock.py
textual/docs/examples/widgets/collapsible.py
textual/docs/examples/widgets/collapsible_custom_symbol.py
textual/docs/examples/widgets/collapsible_nested.py
textual/docs/examples/widgets/content_switcher.py
textual/docs/examples/widgets/content_switcher.tcss
textual/docs/examples/widgets/data_table.py
textual/docs/examples/widgets/data_table_cursors.py
textual/docs/examples/widgets/data_table_fixed.py
textual/docs/examples/widgets/data_table_labels.py
textual/docs/examples/widgets/data_table_renderables.py
textual/docs/examples/widgets/data_table_sort.py
textual/docs/examples/widgets/digits.py
textual/docs/examples/widgets/directory_tree.py
textual/docs/examples/widgets/directory_tree_filtered.py
textual/docs/examples/widgets/footer.py
textual/docs/examples/widgets/header.py
textual/docs/examples/widgets/header_app_title.py
textual/docs/examples/widgets/horizontal_rules.py
textual/docs/examples/widgets/horizontal_rules.tcss
textual/docs/examples/widgets/input.py
textual/docs/examples/widgets/input_types.py
textual/docs/examples/widgets/input_validation.py
textual/docs/examples/widgets/label.py
textual/docs/examples/widgets/link.py
textual/docs/examples/widgets/list_view.py
textual/docs/examples/widgets/list_view.tcss
textual/docs/examples/widgets/loading_indicator.py
textual/docs/examples/widgets/log.py
textual/docs/examples/widgets/markdown.py
textual/docs/examples/widgets/markdown_viewer.py
textual/docs/examples/widgets/masked_input.py
textual/docs/examples/widgets/option_list.tcss
textual/docs/examples/widgets/option_list_options.py
textual/docs/examples/widgets/option_list_strings.py
textual/docs/examples/widgets/option_list_tables.py
textual/docs/examples/widgets/placeholder.py
textual/docs/examples/widgets/placeholder.tcss
textual/docs/examples/widgets/pretty.py
textual/docs/examples/widgets/progress_bar.py
textual/docs/examples/widgets/progress_bar.tcss
textual/docs/examples/widgets/progress_bar_gradient.py
textual/docs/examples/widgets/progress_bar_isolated.py
textual/docs/examples/widgets/progress_bar_isolated_.py
textual/docs/examples/widgets/progress_bar_styled.py
textual/docs/examples/widgets/progress_bar_styled.tcss
textual/docs/examples/widgets/progress_bar_styled_.py
textual/docs/examples/widgets/radio_button.py
textual/docs/examples/widgets/radio_button.tcss
textual/docs/examples/widgets/radio_set.py
textual/docs/examples/widgets/radio_set.tcss
textual/docs/examples/widgets/radio_set_changed.py
textual/docs/examples/widgets/radio_set_changed.tcss
textual/docs/examples/widgets/rich_log.py
textual/docs/examples/widgets/select.tcss
textual/docs/examples/widgets/select_from_values_widget.py
textual/docs/examples/widgets/select_widget.py
textual/docs/examples/widgets/select_widget_no_blank.py
textual/docs/examples/widgets/selection_list.tcss
textual/docs/examples/widgets/selection_list_selected.py
textual/docs/examples/widgets/selection_list_selected.tcss
textual/docs/examples/widgets/selection_list_selections.py
textual/docs/examples/widgets/selection_list_tuples.py
textual/docs/examples/widgets/sparkline.py
textual/docs/examples/widgets/sparkline.tcss
textual/docs/examples/widgets/sparkline_basic.py
textual/docs/examples/widgets/sparkline_basic.tcss
textual/docs/examples/widgets/sparkline_colors.py
textual/docs/examples/widgets/sparkline_colors.tcss
textual/docs/examples/widgets/static.py
textual/docs/examples/widgets/switch.py
textual/docs/examples/widgets/switch.tcss
textual/docs/examples/widgets/tabbed_content.py
textual/docs/examples/widgets/tabbed_content_label_color.py
textual/docs/examples/widgets/tabs.py
textual/docs/examples/widgets/text_area_custom_language.py
textual/docs/examples/widgets/text_area_custom_theme.py
textual/docs/examples/widgets/text_area_example.py
textual/docs/examples/widgets/text_area_extended.py
textual/docs/examples/widgets/text_area_selection.py
textual/docs/examples/widgets/toast.py
textual/docs/examples/widgets/tree.py
textual/docs/examples/widgets/vertical_rules.py
textual/docs/examples/widgets/vertical_rules.tcss
```
