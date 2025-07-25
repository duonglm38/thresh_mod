<script setup>
  import _ from 'lodash';
</script>

<style scoped>
    .child-question {
        display: none;
    }
</style>

<script>
export default {
    props: [
        'edit_state',
        'question',
        'edit_type',

        'parent_show_next_question',
        'set_edit_state',

        'update_edit_state_parent',
        'empty_question_state',
        'question_state',

        'force_update',

        'isRoot',
        'parent_div_name',
        'config'
    ],
    data() { 
        let col_size = 16;
        if ('options' in this.question) {
            switch (this.question.options.length) {
                case 1: col_size = 50; break;
                case 2: col_size = 50; break;
                case 3: col_size = 33; break;
                case 4: col_size = 25; break;
                case 5: col_size = 16; break;
                case 6: col_size = 16; break;
            }
        }

        let div_name;
        if (Boolean(this.isRoot === 'true')) {
            div_name = `question-${this.edit_type.name}-${this.question.name}`
        } else {
            div_name = `${this.parent_div_name}-${this.question.name}`
        }

        return {
            col_size: col_size,
            div_name: div_name
        } 
    },
    methods: {
        has_children() {
            return this.isObject(this.question.options)
        },
        show_next_question(e) {
            let selected_val = e.target.value
            if (this.has_children()) {
                // Hide other children
                for (let i = 0; i < this.question.options.length; i++) {
                    let option = this.question.options[i];
                    let child_div = `.${this.div_name}-${option.name}`
                    $(child_div).hide(300);
                }

                // Show options of selected child
                let child_div = `.${this.div_name}-${selected_val}`
                if (!$(child_div).is(":visible")) {
                    $(child_div).slideDown(300);
                }
            } else {
                // else recurse upwards
                // this.parent_show_next_question(e)
            }
        },
        update_edit_state(val) {
            if (val === "") { val = null }

            if (Boolean(this.isRoot === 'true')) {
                let new_edit_state = _.cloneDeep(this.edit_state);
                new_edit_state[this.edit_type.name][this.question.name] = this.empty_question_state;
                if (this.has_children()) {
                    new_edit_state[this.edit_type.name][this.question.name].val = val;
                } else {
                    new_edit_state[this.edit_type.name][this.question.name] = val;
                }
                this.set_edit_state(new_edit_state);
            } else {
                let new_question_state = _.cloneDeep(this.question_state);
                new_question_state = this.empty_question_state;
                if (this.has_children()) {
                    new_question_state.val = val;
                } else {
                    new_question_state = val;
                }
                this.update_edit_state_parent(this.question.name, new_question_state)
            }
        },
        update_edit_state_child(childName, new_state) {
            let new_question_state = _.cloneDeep(this.question_state);
            new_question_state[childName] = new_state;

            if (Boolean(this.isRoot === 'true')) {
                let new_edit_state = _.cloneDeep(this.edit_state);
                new_edit_state[this.edit_type.name][this.question.name] = this.empty_question_state;
                new_edit_state[this.edit_type.name][this.question.name] = new_question_state;
                this.set_edit_state(new_edit_state);
            } else {
                this.update_edit_state_parent(this.question.name, new_question_state);
            }
        },
        getClassNames() {
            let names = []
            names.push(this.div_name)
            if (Boolean(this.isRoot !== 'true')) {
                names.push('child-question')
            }
            return names
        },
        child_state(child) {
            if (this.has_children() && typeof this.question.options !== 'string') {
                return this.question_state[child.name]
            } else {
                return this.question_state//[child.name] // right?
            }
        },
        empty_child_state(child) {
            if (this.has_children() && typeof this.question.options !== 'string') {
                return this.empty_question_state[child.name]
            } else {
                return this.empty_question_state//[child.name] // right?
            }
        },
        isAnnotated() {
            if (!this.has_children() && this.question_state != null && this.question_state.val == null) { return true }
            if (this.question_state == null || this.question_state.val == null) { return false }
            return true
        }
    },
    computed: {
        isObject() {
            return (variable) => {
                return typeof variable === "object" && variable !== null;
            };
        }
    }
}

</script>

<template>
    <div :class="getClassNames()" :annotated="isAnnotated()" :id="`question_${edit_type.name}_${question.name}`">
        <!-- If the question has children -->
        <div v-if="isObject(question.options)">
            <p class="mb3 b tracked-light">{{ question.question }}</p>
            <div class="tc" >
                <div :class="`column-severity wmin-${col_size}`" v-for="option in question.options" :key="option.id">
                    <input @click="show_next_question" class="checkbox-tools checkbox-tools-severity " type="radio" :name="question.name"
                        :id="`${div_name}-${option.name}`" :value="option.name" @input="update_edit_state($event.target.value)">
                    <label :class="`for-checkbox-tools-severity question-${edit_type.name}`" :for="`${div_name}-${option.name}`">
                        {{ option.label }}
                    </label>
                </div>
            </div>

            <div v-for="child in question.options" :key="child.id">
                <Question :edit_state="edit_state" :question_state="child_state(child)" :empty_question_state="empty_child_state(child)" :question="child" :edit_type="edit_type" :set_edit_state="set_edit_state" 
                    :parent_show_next_question="show_next_question" isRoot=false :update_edit_state_parent="update_edit_state_child"
                    :parent_div_name="div_name" :config="config" />
            </div>
        </div>

        <!-- If the question is likert-3 -->
        <div v-if="question.options === 'likert-3'">
            <p class="mb3 b tracked-light">{{ question.question }}</p>
            <div class="tc">
                <div class="column-severity w-33">
                    <input @click="show_next_question" class="checkbox-tools checkbox-tools-severity " type="radio" :name="`${div_name}-severity`"
                        :id="`${div_name}-severity-1`" value="minor" @input="update_edit_state($event.target.value)">
                    <label :class="`for-checkbox-tools-severity question-${edit_type.name}`" :for="`${div_name}-severity-1`">
                        {{ config.interface_text.questions.likert_1 }}
                    </label>
                </div>
                <div class="column-severity w-33">
                    <input @click="show_next_question" class="checkbox-tools checkbox-tools-severity " type="radio" :name="`${div_name}-severity`"
                        :id="`${div_name}-severity-2`" value="somewhat" @input="update_edit_state($event.target.value)">
                    <label :class="`for-checkbox-tools-severity question-${edit_type.name}`" :for="`${div_name}-severity-2`">
                        {{ config.interface_text.questions.likert_2 }}
                    </label>
                </div>
                <div class="column-severity w-33">
                    <input @click="show_next_question" class="checkbox-tools checkbox-tools-severity " type="radio" :name="`${div_name}-severity`"
                        :id="`${div_name}-severity-3`" value="a lot" @input="update_edit_state($event.target.value)">
                    <label :class="`for-checkbox-tools-severity question-${edit_type.name}`" :for="`${div_name}-severity-3`">
                        {{ config.interface_text.questions.likert_3 }}
                    </label>
                </div>
            </div>
        </div>

        <!-- If the question is binary -->
        <div v-if="question.options === 'binary'">
            <p class="mt0 pt2 mb3 b tracked-light"> {{ question.question }}
                <input class="checkbox-tools-yes-no" type="radio" :name="`${div_name}-yes-no`"
                    :id="`${div_name}-yes`" value="yes" @input="update_edit_state($event.target.value)">
                <label :class="`normal for-checkbox-tools-yes-no question-${edit_type.name}`" :for="`${div_name}-yes`">
                    {{ config.interface_text.questions.binary_yes }}
                </label>
                <input class="checkbox-tools-yes-no" type="radio" :name="`${div_name}-yes-no`"
                    :id="`${div_name}-no`" value="no" @input="update_edit_state($event.target.value)">
                <label :class="`normal for-checkbox-tools-yes-no question-${edit_type.name}`" :for="`${div_name}-no`">
                    {{ config.interface_text.questions.binary_no }}
                </label>
            </p>
        </div>

        <!-- If the question is an inline text box -->
        <div v-if="question.options === 'textbox'" class="flex items-center mt2">
            <p class="mb3 b tracked-light">{{ question.question }}</p>
            <input type="text" class="question-textbox" :name="`${div_name}-textbox`" :id="`${div_name}`" 
                :placeholder="config.interface_text.questions.textbox_placeholder" @input="update_edit_state($event.target.value)"
                :class="`db border-box hover-black ba b--black-20 pa2 br2 ml3 flex-auto`" />
            <label :class="`normal for-question-textbox question-${edit_type.name}`" :for="`${div_name}`"></label>
            
        </div>

        <!-- If the question is an open-ended text area -->
        <div v-if="question.options === 'textarea'">
            <p class="mt0 pt2 mb3 b tracked-light"> {{ question.question }}</p>

            <textarea class="question-textarea" :name="`${div_name}-textarea`" :id="`${div_name}`" :placeholder="config.interface_text.questions.textbox_placeholder" 
                @input="update_edit_state($event.target.value)" :class="`db border-box hover-black w-100 ba b--black-20 pa2 br2 mb2`" />
            <label :class="`normal for-question-textarea question-${edit_type.name}`" :for="`${div_name}`"></label>
        </div>
    </div>
</template>