<script setup>
    import _ from 'lodash';
    import SourceSent from "./SourceSent.vue";
    import TargetSent from "../hitbox/TargetSent.vue";
</script>

<script>
export default {
    data() {
        return {

        }
    },
    props: [
        'hits_data',
        'current_hit',
        'edits_dict',
        'selected_edits_html',
        'selected_edits',
        'set_selected_edits',
        'set_span_text',
        'set_span_indices',
        'set_span_category',
        'remove_selected',
        'lines',
        'set_lines',
        'set_edit_html',
        'hit_box_config',
        'config',

        'sent_type',

        'selected_state',
    ],
    methods: {
        process_edit_list(edits, sent_type) {
            edits = edits.map(edit => {
                let flattened_edits = []
                if (edit.hasOwnProperty('constituent_edits')) {
                    for (const const_edit of edit['constituent_edits']) {
                        if (const_edit.hasOwnProperty(sent_type)) {
                            for (const in_idx of const_edit[sent_type]) {
                                flattened_edits.push({
                                    [sent_type]: in_idx,
                                    'category': edit['category'],
                                    'child_category': const_edit['category'],
                                    'id': edit['id'],
                                    'child_id': const_edit['id'],
                                    'annotation': edit['annotation']
                                })
                            }
                        }
                    }
                }
                if (edit.hasOwnProperty(sent_type)) {
                    for (const in_idx of edit[sent_type]) {
                        flattened_edits.push({
                            [sent_type]: in_idx,
                            'category': edit['category'],
                            'id': edit['id'],
                            'annotation': edit['annotation']
                        })
                    }
                }
                return flattened_edits
            })
            edits = [].concat(...edits); // flatten list
            edits.sort(function(a, b) {
                return a[sent_type][0] - b[sent_type][0] || a[sent_type][1] - b[sent_type][1];
            });
            return edits
        },
        hasAnnotations(edit) {
            return (("annotation" in edit) && edit["annotation"] != null && edit["annotation"] != "")  || (this.config.disable && Object.values(this.config.disable).includes('annotation'))
        },
        is_selected(edit, sent_type, selected_category) {
            let whether_is_selected
            const selected_idx = this.get_selected_index(sent_type)
            if (this.multi_select_enabled(sent_type)) {
                whether_is_selected = selected_idx.some(span => edit[sent_type][0] == span[0] && edit[sent_type][1] == span[1] && edit['category'] == selected_category);
                // for (let j = 0; j < selected_idx.length; j++) {
                //     if (edit[sent_type][0] == selected_idx[j][0] && edit[sent_type][1] == selected_idx[j][1] && edit['category'] == selected_category) {
                //         whether_is_selected = true
                //         break
                //     }
                // }
            } else {
                whether_is_selected = edit[sent_type][0] == selected_idx[0] && edit[sent_type][1] == selected_idx[1] && edit['category'] == selected_category
            }
            return whether_is_selected
        },
        get_selected_index(sent_type) {
            if (sent_type == 'input_idx') {
                return _.cloneDeep(this.selected_state.source_idx)
            }
            if (sent_type == 'output_idx') {
                return _.cloneDeep(this.selected_state.target_idx)
            }
            return undefined
        },
        multi_select_enabled(sent_type) {
            if (sent_type == 'input_idx') {
                return this.hit_box_config.enable_multi_select_source_sentence
            }
            if (sent_type == 'output_idx') {
                return this.hit_box_config.enable_multi_select_target_sentence
            }
            return undefined
        },
        hover_span(e) {
            if ($(".quality-selection").is(":visible")) {
                return
            }

            const category = e.target.dataset.category
            const id = e.target.dataset.id
            const real_id = id.split("-")[1]

            let color_code, color_class;
            if (e.target.classList.contains(`border-${category}-light`)) {
                color_code = `bg-${category}-light`
                color_class = "rgba(173, 197, 250, 1.0)"
            } else {
                color_code = `bg-${category}`
                color_class = "rgba(33, 134, 235, 1.0)"
            }

            let spans = $(`.${category}[data-id=${id}]`)
            let below_spans= $(`.${category}_below[data-id=${id}]`)
            spans.addClass(`white ${color_code}`)
            below_spans.addClass(`white ${color_code}`)
            below_spans.removeClass(`txt-${category} txt-${category}-light`)

            try {
                if (category == 'substitution') {
                    this.lines[category][real_id].color = color_class
                }
            } catch (e) { console.log(e) }
        },
        un_hover_span(e) {
            if ($(".quality-selection").is(":visible")) {
                return
            }

            const category = e.target.dataset.category
            const id = e.target.dataset.id
            const real_id = id.split("-")[1]

            let color_code, color_class;
            if (e.target.classList.contains(`border-${category}-light`)) {
                color_code = "rgba(173, 197, 250, 0.4)"
                color_class = `txt-${category}-light`
            } else {
                color_code = "rgba(33, 134, 235, 0.46)"
                color_class = `txt-${category}`
            }

            let spans = $(`.${category}[data-id=${id}]`)
            let below_spans= $(`.${category}_below[data-id=${id}]`)
            below_spans.addClass(color_class)
            spans.removeClass(`white bg-${category} bg-${category}-light`)
            below_spans.removeClass(`white bg-${category} bg-${category}-light`)

            try {
                if (category == 'substitution') {
                    this.lines[category][real_id].color = color_code
                }
            } catch (e) { console.log(e) }
        },
        getEditConfig(category) {
            return this.config['edits'].find(function(entry) {
                return entry['name'] === category;
            });
        },
        click_span(e) {
            const edits_dict = this.edits_dict
            const category = e.target.dataset.category
            const id = e.target.dataset.id
            const real_id = id.split("-")[1]

            if ($(".quality-selection").is(":visible")) {
                if (this.getEditConfig(category)['type'] && this.getEditConfig(category)['type'] == 'composite') { return }

                const selected_span = this.hits_data[this.current_hit - 1]['edits'].find(function(entry) {
                    return entry['category'] === category && entry['id'] === parseInt(real_id);
                });

                // Has this been selected before?
                const exists = this.selected_edits.find(function(entry) {
                    return entry['category'] === category && entry['id'] === parseInt(real_id);
                }) === undefined ? false : true;
                
                // Rules for selecting split signs
                if ($("input[name=edit_cotegory]:checked").val() == 'split') {
                    let normal_id = parseInt(real_id) + 1
                    if (e.target.classList.contains(`split-sign`)) {
                        if (normal_id == 1) {
                            this.selected_state.selected_split = `the 1st split`
                        } else if (normal_id == 2) {
                            this.selected_state.selected_split = `the 2nd split`
                        } else if (normal_id == 3) {
                            this.selected_state.selected_split = `the 3rd split`
                        } else {
                            this.selected_state.selected_split = `the ${normal_id}th split`
                        }
                        this.selected_state.split_id = parseInt(real_id)
                    } 
                }

                let new_selected_edits = _.cloneDeep(this.selected_edits);
                if (!exists) {
                    // Select edit
                    new_selected_edits.push(selected_span)
                } else {
                    // Unselect edit
                    new_selected_edits = new_selected_edits.filter(o => o.category !== category || o.id !== parseInt(real_id));
                }
                
                // Render selected edits
                let new_edit_html = ""
                for (const edit of new_selected_edits) {
                    new_edit_html += this.render_selected_constituent_edit(edit)
                }
                
                this.set_selected_edits(new_selected_edits)
                this.set_edit_html(new_edit_html)
            } else {
                // This provides the default behavior: simply triggering another action
                $(`.annotation-icon[data-id=${id}]`).click()
            }
        },
        render_selected_constituent_edit(edit) {
            // TODO: This is duplicate code from EditList's render_edit_text, need to clean this up
            let edit_html = ''
            const key = edit.category
            
            if (this.getEditConfig(key)['type'] == 'multi_span') {
                edit_html += `<span class="edit-type txt-${key}">substitute </span>`;
                let source_spans = edit['input_idx']
                for (let j = 0; j < source_spans.length; j++) {
                    if (j != 0) {
                        edit_html += `<span class="edit-type txt-${key}"> and </span>`;
                    }
                    edit_html += `<span class="pa1 edit-text br-pill-ns txt-${key} border-${key}-all">
                        &nbsp${this.hits_data[this.current_hit - 1].source.substring(source_spans[j][0], source_spans[j][1])}&nbsp</span>`;
                }

                edit_html += `<span class="edit-type txt-${key}"> with </span>`;

                let target_spans = edit['output_idx']
                for (let j = 0; j < target_spans.length; j++) {
                    if (j != 0) {
                        edit_html += `<span class="edit-type txt-${key}"> and </span>`;
                    }
                    edit_html += `<span class="pa1 edit-text br-pill-ns txt-${key} border-${key}-all">
                        &nbsp${this.hits_data[this.current_hit - 1].target.substring(target_spans[j][0], target_spans[j][1])}&nbsp</span>`;
                }
                edit_html += ",&nbsp&nbsp";
            } else {
                if (edit.hasOwnProperty('input_idx')) {
                    let in_span = edit['input_idx'][0]
                    edit_html += `<span class="edit-type txt-${key}">${key} </span>
                        <span class="pa1 edit-text br-pill-ns txt-${key} border-${key}-all">
                            &nbsp${this.hits_data[this.current_hit - 1].source.substring(in_span[0], in_span[1])}&nbsp</span>,&nbsp&nbsp`;
                }
                if (edit.hasOwnProperty('output_idx')) {
                    let out_span = edit['output_idx'][0]
                    edit_html += `<span class="edit-type txt-${key}">${key} </span>
                        <span class="pa1 edit-text br-pill-ns txt-${key} border-${key}-all">
                            &nbsp${this.hits_data[this.current_hit - 1].target.substring(out_span[0], out_span[1])}&nbsp</span>,&nbsp&nbsp`;
                }
            }

            return edit_html
        },
        handle_tokenization_rendering() { 
            if (this.config.tokenization && this.config.tokenization == 'tokenized') {
                $('span#source-sentence, span#target-sentence, .edit-text, .selected-span-text').addClass('hide-tokenization-chars')
            } else {
                $('span#source-sentence, span#target-sentence, .edit-text, .selected-span-text').removeClass('hide-tokenization-chars')
            }
        },
        _buildSpanOpenTag(edit, sent_type, span_class, selected_category, includes_selection) {
            let light = !this.hasAnnotations(edit) ? "-light" : "";
            let composite_info = edit.hasOwnProperty('child_category') ? `data-childcategory=${edit['child_category']} data-childid=${edit['child_id']}` : "";

            // --- NEW: Case 0: The span is a pre-defined focus area. ---
            // Render it as a simple, non-interactive background highlight.
            if (edit['category'] === 'focus-area') {
                return `<span class="focus-area-highlight">`;
            }

            // Case 1: The span is the one currently being selected by the user.
            // Render it as a simple, non-interactive highlighted area.
            if (includes_selection && this.is_selected(edit, sent_type, selected_category)) {
                return `<span @mouseover.stop @mouseout.stop class="bg-${edit['category']}-light span">`;
            }

            // Case 2: A standard, interactive annotation span.
            else {
                return `<span @click="click_span" @mouseover="hover_span" @mouseout="un_hover_span" class="${edit['category']} border-${edit['category']}${light} pointer span ${span_class}" data-category="${edit['category']}" data-id="${edit['category']}-${edit['id']}" ${composite_info}>`;
            }
        },
        render_sentence(sent, sent_type, span_class, selected_category) {
            // Get the data for the current hit once to avoid repeated lookups
            const current_hit_data = this.hits_data[this.current_hit - 1];
            let hit_edits = _.cloneDeep(current_hit_data.edits);
            const includes_selection = selected_category != null && selected_category != '';

            // --- NEW (REVISED): Add focus areas from the current hit's data ---
            // Determine the correct key for focus areas based on the sentence type.
            const focus_area_key = (sent_type === 'input_idx') 
                                    ? 'source_focus_areas' 
                                    : 'target_focus_areas';

            // Get the specific focus areas array from the current hit's data.
            const focus_areas = current_hit_data[focus_area_key];

            // Check if focus areas exist for this sentence type and add them as special edits.
            if (focus_areas && Array.isArray(focus_areas)) {
                focus_areas.forEach((area, index) => {
                    hit_edits.push({
                        "category": "focus-area", // A special, non-interactive category
                        "id": `focus-${sent_type}-${index}`, // A more specific unique ID
                        [sent_type]: [area],        // 'area' should be [start, end]
                    });
                });
            }
            // --- END NEW (REVISED) CODE ---

            if (includes_selection) {
                const selected_idx = this.get_selected_index(sent_type);
                if (this.multi_select_enabled(sent_type) && Array.isArray(selected_idx[0])) {
                    for (let i = 0; i < selected_idx.length; i++) {
                        hit_edits.push({
                            "category": selected_category,
                            [sent_type]: [selected_idx[i]],
                        });
                    }
                } else {
                    hit_edits.push({
                        "category": selected_category,
                        [sent_type]: [selected_idx]
                    });
                }
            }
            const allEdits = this.process_edit_list(hit_edits, sent_type);

            if (allEdits.length === 0) {
                return sent; // No edits, just return the plain sentence.
            }

            // --- STAGE 2: SEGMENTATION ALGORITHM ---
            // NO CHANGES ARE NEEDED HERE. It will work perfectly with the new focus areas.
            const points = new Set([0, sent.length]);
            allEdits.forEach(edit => {
                points.add(edit[sent_type][0]);
                points.add(edit[sent_type][1]);
            });
            const splitPoints = Array.from(points).sort((a, b) => a - b);
            let sentence_html = '';

            for (let i = 0; i < splitPoints.length - 1; i++) {
                const start = splitPoints[i];
                const end = splitPoints[i + 1];

                if (start >= end) {
                    continue;
                }

                const segmentText = sent.substring(start, end);

                const enclosingEdits = allEdits.filter(edit =>
                    edit[sent_type][0] <= start && edit[sent_type][1] >= end
                );

                enclosingEdits.sort((a, b) => {
                    const a_start = a[sent_type][0];
                    const b_start = b[sent_type][0];
                    if (a_start !== b_start) {
                        return b_start - a_start;
                    }
                    const a_end = a[sent_type][1];
                    const b_end = b[sent_type][1];
                    return a_end - b_end;
                });

                let segmentHtml = segmentText;
                for (const edit of enclosingEdits) {
                    const openTag = this._buildSpanOpenTag(edit, sent_type, span_class, selected_category, includes_selection);
                    segmentHtml = `${openTag}${segmentHtml}</span>`;
                }

                sentence_html += segmentHtml;
            }

            return sentence_html;
        },
    },
}
</script>

<template>
    <div>
        <template v-if="sent_type === 'source'">
            <span class="f4 lh-paras context-span" v-if="hits_data && hits_data[current_hit - 1] && hits_data[current_hit - 1].source_context_before">
                {{ hits_data[current_hit - 1].source_context_before }}&nbsp;
            </span>
            
            <SourceSent sent_type="source" v-bind="$props" :remove_selected="remove_selected" :process_edit_list="process_edit_list" :handle_tokenization_rendering="handle_tokenization_rendering"
            :hasAnnotations="hasAnnotations" :is_selected="is_selected" :get_selected_index="get_selected_index" :multi_select_enabled="multi_select_enabled" 
            :render_sentence="render_sentence" :click_span="click_span" :hover_span="hover_span" :un_hover_span="un_hover_span" :config="config" />
        
            <span class="f4 lh-paras context-span" v-if="hits_data && hits_data[current_hit - 1] && hits_data[current_hit - 1].source_context_after">
                &nbsp;{{ hits_data[current_hit - 1].source_context_after }}
            </span>
        </template>
        <template v-else-if="sent_type === 'target'">
            <span class="f4 lh-paras context-span" v-if="hits_data && hits_data[current_hit - 1] && hits_data[current_hit - 1].target_context_before">
                {{ hits_data[current_hit - 1].target_context_before }}&nbsp;
            </span>

            <TargetSent sent_type="target" v-bind="$props" :remove_selected="remove_selected" :process_edit_list="process_edit_list" :handle_tokenization_rendering="handle_tokenization_rendering"
            :hasAnnotations="hasAnnotations" :is_selected="is_selected" :get_selected_index="get_selected_index" :multi_select_enabled="multi_select_enabled" 
            :render_sentence="render_sentence" :click_span="click_span" :hover_span="hover_span" :un_hover_span="un_hover_span" :config="config" />
        
            <span class="f4 lh-paras context-span" v-if="hits_data && hits_data[current_hit - 1] && hits_data[current_hit - 1].target_context_after">
                &nbsp;{{ hits_data[current_hit - 1].target_context_after }}
            </span>
        </template>
    </div>
</template>
