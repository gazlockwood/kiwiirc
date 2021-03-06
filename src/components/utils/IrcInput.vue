<template>
    <div class="kiwi-ircinput">
        <div
            class="kiwi-ircinput-editor"
            contenteditable="true"
            role="textbox"
            :placeholder="placeholder"
            ref="editor"
            @keypress="updateValueProps(); $emit('keypress', $event)"
            @keydown="updateValueProps(); $emit('keydown', $event)"
            @keyup="updateValueProps(); $emit('keyup', $event)"
            @textInput="updateValueProps(); onTextInput($event); $emit('textInput', $event)"
            @mouseup="updateValueProps();"
            @click="$emit('click', $event)"
        ></div>
    </div>
</template>

<script>

import htmlparser from 'htmlparser2';

export default {
    data: function data() {
        return {
            text_value: '',
            current_el: null,
            current_el_pos: 0,
            current_range: null,
            default_colour: null,
            code_map: Object.create(null),
        };
    },
    props: ['value', 'placeholder'],
    computed: {
        editor: function editor() {
            return this.$refs.editor;
        },
    },
    watch: {
        value: function watchValue(newVal) {
            // When we trigger input events Vue updates the value prop for us. If the updated value
            // matches what we already have then the change came from the editor so we don't need
            // to update it. But if it differs then the change came from elsewhere so we must
            // update.
            if (this.$refs.editor.innerHTML !== newVal) {
                this.$refs.editor.innerHTML = newVal;
            }

            this.text_value = this.$refs.editor.innerText;
        },
    },
    methods: {
        onTextInput: function onTextInput(event) {
            // Mobile devices trigger a textInput event for things such as autocompletion
            // and sugegsted words. Unfortunately they end with a return character which
            // is not what we expect, so prevent the original event from inserting anything
            // and manually place it in with the current word.
            if (event.data[event.data.length - 1] === '\n') {
                event.preventDefault();
                this.setCurrentWord(event.data.trim());
            }
        },
        updateValueProps: function updateValueProps() {
            let selection = window.getSelection();
            this.current_el_pos = selection.anchorOffset;
            this.current_el = selection.anchorNode;

            let range = null;
            let currentRange = selection.getRangeAt(0);
            if (currentRange) {
                range = currentRange.cloneRange();
                range.selectNodeContents(this.$refs.editor);
                range.setEnd(currentRange.startContainer, currentRange.startOffset);
            }

            let start = range.toString().length;
            this.current_range = [
                start,
                start + currentRange.toString().length,
            ];

            this.maybeEmitInput();
        },
        maybeEmitInput: function maybeEmitInput() {
            let currentHtml = this.$refs.editor.innerHTML;
            if (this.value !== currentHtml) {
                // Vuejs v-model stuff picks up this event and sets the value prop
                // with the value
                this.$emit('input', currentHtml);
            }
        },
        buildIrcText: function buildIrcText() {
            let source = this.$refs.editor.innerHTML;
            let textValue = '';

            let parser = new htmlparser.Parser({
                onopentag: (name, attribs) => {
                    let codeLookup = '';
                    if (attribs.style) {
                        let match = attribs.style.match(/color: ([^;]+)/);
                        if (match) {
                            codeLookup = match[1];
                            if (this.code_map[codeLookup]) {
                                textValue += '\x03' + this.code_map[codeLookup];
                            }
                        }
                    }
                    if (attribs.src && this.code_map[attribs.src]) {
                        textValue += this.code_map[attribs.src];
                    }
                },
                ontext: text => {
                    textValue += text;
                },
                onclosetag: tagName => {
                    if (tagName === 'span') {
                        textValue += '\x03';
                    }
                },
            }, {
                decodeEntities: true,
            });

            /* eslint max-len: off */
            parser.write(source);
            parser.end();

            return textValue;
        },
        reset: function reset(rawHtml) {
            this.$refs.editor.innerHTML = rawHtml || '';

            // Firefox inserts a <br> on empty contenteditables after it's been reset. But that
            // fucks up the placeholder :empty CSS selector we use. So just remove it.
            let br = this.$refs.editor.querySelector('br');
            if (br) {
                br.remove();
            }

            if (this.default_colour) {
                this.focus();
                this.setColour(this.default_colour.code, this.default_colour.colour);
            }

            this.updateValueProps();
        },
        resetStyles: function resetStyles() {
            this.focus();
            document.execCommand('styleWithCSS', false, true);
            document.execCommand('selectAll', false, null);
            document.execCommand('removeFormat', false, null);
            this.default_colour = null;
        },
        setColour: function setColour(code, colour) {
            // If no current text selection, set this colour as the default colour for
            // future messages too
            let range = window.getSelection().getRangeAt(0);
            if (range && range.collapsed) {
                this.default_colour = {
                    code,
                    colour,
                };
            }

            this.focus();
            document.execCommand('styleWithCSS', false, true);
            document.execCommand('foreColor', false, colour);

            this.code_map[colour] = code;
            this.updateValueProps();
        },
        addImg: function addImg(code, url) {
            this.focus();
            document.execCommand('styleWithCSS', false, true);
            document.execCommand('insertImage', false, url);

            this.code_map[url] = code;
            this.updateValueProps();
        },

        // Insert some text at the current position
        insertText: function insertText(text) {
            let el = this.current_el;
            let pos = this.current_el_pos;
            let val = el.textContent;
            el.textContent = val.substr(0, pos) + text + val.substr(pos);
        },

        // Replace the word at the current position with another
        setCurrentWord: function setCurrentWord(text, keepPosition) {
            let el = this.current_el;
            let pos = this.current_el_pos;
            let val = el.textContent;

            let startVal = val.substr(0, pos);
            let space = startVal.lastIndexOf(' ');
            if (space === -1) {
                space = 0;
            } else {
                // include the space after the word
                space++;
            }
            let startPos = space;

            let endVal = val.substr(pos);
            space = endVal.indexOf(' ');
            if (space === -1) space = endVal.length;
            let endPos = space;

            el.textContent = startVal.substr(0, startPos) + text + endVal.substr(endPos);

            // Move the caret after the word
            let range = document.createRange();
            if (keepPosition) {
                range.setStart(el, pos);
                range.setEnd(el, pos);
            } else {
                range.setStart(el, startPos + text.length);
                range.setEnd(el, startPos + text.length);
            }

            let sel = window.getSelection();
            sel.removeAllRanges();
            sel.addRange(range);
        },

        getCurrentWord: function getCurrentWord() {
            let el = this.current_el;
            let pos = this.current_el_pos;
            let val = el.textContent;

            let startVal = val.substr(0, pos);
            let space = startVal.lastIndexOf(' ');
            if (space === -1) {
                space = 0;
            } else {
                // include the space after the word
                space++;
            }
            let startPos = space;

            space = val.indexOf(' ', startPos);
            if (space === -1) space = val.length;
            let endPos = space;

            return {
                word: val.substr(startPos, endPos),
                position: pos - startPos,
            };
        },

        getRawText: function getRawText() {
            return this.$refs.editor.innerText;
        },

        // Focus the editable div and move the caret to the end
        focus: function focus() {
            if (this.current_range) {
                let selection = window.getSelection();
                let savedSel = this.current_range;

                let charIndex = 0;
                let range = document.createRange();
                range.setStart(this.$refs.editor, 0);
                range.collapse(true);
                let nodeStack = [this.$refs.editor];
                let node = null;
                let foundStart = false;
                let stop = false;

                while (!stop && (node = nodeStack.pop())) {
                    if (node.nodeType === 3) {
                        let nextCharIndex = charIndex + node.length;
                        if (!foundStart && savedSel[0] >= charIndex && savedSel[0] <= nextCharIndex) {
                            range.setStart(node, savedSel[0] - charIndex);
                            foundStart = true;
                        }
                        if (foundStart && savedSel[1] >= charIndex && savedSel[1] <= nextCharIndex) {
                            range.setEnd(node, savedSel[1] - charIndex);
                            stop = true;
                        }
                        charIndex = nextCharIndex;
                    } else {
                        let i = node.childNodes.length;
                        while (i--) {
                            nodeStack.push(node.childNodes[i]);
                        }
                    }
                }

                // Firefox needs the manual focus() call
                this.$refs.editor.focus();
                selection.removeAllRanges();
                selection.addRange(range);
            } else {
                this.$refs.editor.focus();
            }
        },
    },
};
</script>

<style>

.kiwi-ircinput {
    box-sizing: border-box;
    position: relative;
    overflow: visible;
    z-index: 1;
}
.kiwi-ircinput-editor {
    white-space: nowrap;
    overflow-x: hidden;
    outline: none;
}
.kiwi-ircinput-editor:empty:not(:focus):before {
    content: attr(placeholder);
    cursor: text;
}
.kiwi-ircinput-editor img {
    height: 1em;
}

</style>