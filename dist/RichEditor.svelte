<script>
    import { onMount } from 'svelte';
    import { tick } from 'svelte';
    import { html, object } from './stores';

    /** @type {import('./types').RichObject} */
    export let content;

    /** @type {HTMLDivElement | null} */
    let editor = null;

    /** @type {Selection | null}*/
    let selection = null;

    /** @type {import('./types').RichSelectionRange | undefined}*/
    let richSelection;

    $: if (liveObject) html.set(objectToHtml(liveObject));
    $: if (richObject) object.set(richObject);

    /** @type {import('./types').RichObject} */
    let richObject;

    /** @type {import('./types').RichObject} */
    let liveObject;

    function handleSelectionChange() {
        selection = document.getSelection();

        // Anchor and focus need to be in editor (anchor and focus nodes are start and end nodes of selection)
        if (editor && selection && editor.contains(selection.focusNode) && editor.contains(selection.anchorNode)) {
            const length = selection.toString().length;

            let range = selection.getRangeAt(0);

            let startNode = selection.anchorNode;
            let startChildNode;

            while (startNode?.parentElement && startNode?.parentElement !== editor) {
                startChildNode = startNode;
                startNode = startNode?.parentElement;
            }

            const startIndex = Array.prototype.indexOf.call(startNode?.parentNode?.children, startNode);
            const startChildIndex = Array.prototype.indexOf.call(startNode?.childNodes, startChildNode);

            let totalOffset = 0;
            if (startChildIndex > 0) {
                for (let i = 0; i < startChildIndex; i++) {
                    totalOffset += editor.children[startIndex].childNodes[i].textContent.length;
                }
                totalOffset += range.startOffset;
            } else {
                totalOffset = range.startOffset;
            }

            if (selection.anchorNode === selection.focusNode) {
                richSelection = {
                    start: {
                        index: startIndex,
                        childIndex: startChildIndex,
                        offset: range.startOffset,
                    },
                    end: {
                        index: startIndex,
                        childIndex: startChildIndex,
                        offset: range.endOffset,
                    },
                    length,
                    totalOffset,
                };
            } else {
                let endNode = selection.focusNode;
                let endChildNode;
                // Find parent node
                while (endNode?.parentElement && endNode?.parentElement !== editor) {
                    endChildNode = endNode;
                    endNode = endNode?.parentNode;
                }

                const endIndex = Array.prototype.indexOf.call(endNode?.parentNode?.children, endNode);
                const endChildIndex = Array.prototype.indexOf.call(endNode?.childNodes, endChildNode);
                if (endIndex < startIndex || (endIndex === startIndex && endChildIndex < startChildIndex)) {
                    let totalOffset = 0;
                    if (endChildIndex > 0) {
                        for (let i = 0; i < endChildIndex; i++) {
                            totalOffset += editor.children[endIndex].childNodes[i].textContent.length;
                        }
                        totalOffset += range.startOffset;
                    } else {
                        totalOffset = range.startOffset;
                    }
                    richSelection = {
                        start: {
                            index: endIndex,
                            childIndex: endChildIndex,
                            offset: range.startOffset,
                        },
                        end: {
                            index: startIndex,
                            childIndex: startChildIndex,
                            offset: range.endOffset,
                        },
                        length,
                        totalOffset,
                    };
                } else {
                    richSelection = {
                        start: {
                            index: startIndex,
                            childIndex: startChildIndex,
                            offset: range.startOffset,
                        },
                        end: {
                            index: endIndex,
                            childIndex: endChildIndex,
                            offset: range.endOffset,
                        },
                        length,
                        totalOffset,
                    };
                }
            }
        } else {
            richSelection = undefined;
        }
    }

    function setSelection() {
        if (richSelection && editor) {
            let offset = richSelection.totalOffset;
            let length = richSelection.length;
            for (let i = 0; i < editor.children[richSelection.start.index].childNodes.length; i++) {
                if (editor.children[richSelection.start.index].childNodes[i].textContent.length > offset) {
                    richSelection.start.childIndex = i;
                    richSelection.start.offset = offset;
                    if (offset + length > editor.children[richSelection.start.index].childNodes[i].textContent.length) {
                        length -= editor.children[richSelection.start.index].childNodes[i].textContent.length - offset;
                    } else {
                        richSelection.end.childIndex = i;
                        richSelection.end.offset = offset + length;
                        break;
                    }
                    i++;
                    while (
                        i < editor.children[richSelection.start.index].childNodes.length &&
                        editor.children[richSelection.start.index].childNodes[i].textContent.length < length
                    ) {
                        length -= editor.children[richSelection.start.index].childNodes[i].textContent.length;
                        i++;
                    }
                    richSelection.end.childIndex = i;
                    richSelection.end.offset = length;
                    break;
                }
                offset -= editor.children[richSelection.start.index].childNodes[i].textContent.length;
            }

            let startNode = editor?.children[richSelection.start.index].childNodes[richSelection.start.childIndex];
            while (startNode?.hasChildNodes() && startNode.firstChild) {
                startNode = startNode.firstChild;
            }

            let endNode = editor?.children[richSelection.end.index].childNodes[richSelection.end.childIndex];
            while (endNode?.hasChildNodes() && endNode.firstChild) {
                endNode = endNode.firstChild;
            }
            if (startNode && endNode) {
                let newRange = document.createRange();
                newRange.setStart(startNode, richSelection.start.offset);
                newRange.setEnd(endNode, richSelection.end.offset);
                selection?.removeAllRanges();
                selection?.addRange(newRange);
            }
        }
    }

    /** @param {import('./types').TextNode['type']} type */
    function updateNodeType(type) {
        if (!richSelection) {
            return;
        }
        if (richSelection.start.index === richSelection.end.index) {
            richObject[richSelection.start.index].type = type;
        } else {
            for (let i = richSelection.start.index; i <= richSelection.end.index; i++) {
                richObject[i].type = type;
            }
        }

        liveObject = richObject;
        // To update in view
        richObject = richObject;
    }

    /**
     * @param {import('./types').HtmlFormat} action
     * @param {string | undefined} attribute
     */
    async function updateStyle(action, attribute = undefined) {
        if (!richSelection) {
            return;
        }
        if (richSelection.start.index === richSelection.end.index) {
            updateObject(action, richSelection.totalOffset, richSelection.length, richSelection.start.index, attribute);
        } else {
            // Update first object
            updateObject(
                action,
                richSelection.totalOffset,
                richObject[richSelection.start.index].text.length - richSelection.start.offset,
                richSelection.start.index,
                attribute,
            );

            // Update middle objects
            for (let i = richSelection.start.index + 1; i < richSelection.end.index; i++) {
                updateObject(action, 0, richObject[i].text.length, i, attribute);
            }

            // Update last object
            updateObject(action, 0, richSelection.end.offset, richSelection.end.index, attribute);
        }
        cleanObject();
        liveObject = richObject;

        // To update in view
        richObject = richObject;
        await tick();
        setSelection();
    }

    /** @param {import('./types').HtmlFormat} action
     * @param {number} totalOffset
     * @param {number} length
     * @param {number} objIndex
     * @param {string | undefined} attribute
     */
    function updateObject(action, totalOffset, length, objIndex, attribute = undefined) {
        let updateObject = richObject[objIndex];

        if (!updateObject.format) {
            // No formatting yet -> Create format object
            updateObject.format = [
                {
                    start: totalOffset,
                    end: totalOffset + length,
                    [action]: attribute ?? true,
                },
            ];
            return;
        }

        let insertIndex = 0;
        while (
            insertIndex < updateObject.format.length &&
            updateObject.format[insertIndex] &&
            updateObject.format[insertIndex]?.start < totalOffset &&
            updateObject.format[insertIndex]?.end < totalOffset
        ) {
            insertIndex++;
        }

        // Create new at the end
        if (insertIndex === updateObject.format.length) {
            updateObject.format.push({
                start: totalOffset,
                end: totalOffset + length,
                [action]: attribute ?? true,
            });
            return;
        }

        // Create new at the beginning or between two format objects
        if (updateObject.format[insertIndex].start > totalOffset + length) {
            updateObject.format.splice(insertIndex, 0, {
                start: totalOffset,
                end: totalOffset + length,
                [action]: attribute ?? true,
            });
            return;
        }

        let actionValue = !(action in updateObject.format[insertIndex]) && !updateObject.format[insertIndex][action];
        if (updateObject.format[insertIndex].start <= totalOffset && updateObject.format[insertIndex].end >= totalOffset + length) {
            if (updateObject.format[insertIndex]?.start === totalOffset && updateObject.format[insertIndex]?.end === totalOffset + length) {
                // Selection (range) is same as existing element
                // Format object stays the same
                if (actionValue) {
                    updateObject.format[insertIndex][action] = attribute ?? actionValue;
                } else {
                    delete updateObject.format[insertIndex][action];
                }
                return;
            } else if (updateObject.format[insertIndex]?.start === totalOffset) {
                // Format object needs to be split at end
                let oldFormat = { ...updateObject.format[insertIndex] };
                oldFormat.start = updateObject.format[insertIndex].end = totalOffset + length;

                if (actionValue) {
                    updateObject.format[insertIndex][action] = attribute ?? actionValue;
                    updateObject.format.splice(insertIndex + 1, 0, oldFormat);
                } else {
                    delete updateObject.format[insertIndex][action];
                    updateObject.format.splice(insertIndex + 1, 0, oldFormat);
                }
            } else if (updateObject.format[insertIndex]?.end === totalOffset + length) {
                // Format object needs to be split at start
                let oldFormat = { ...updateObject.format[insertIndex] };
                oldFormat.end = updateObject.format[insertIndex].start = totalOffset;

                if (actionValue) {
                    updateObject.format[insertIndex][action] = attribute ?? actionValue;
                    updateObject.format.splice(insertIndex, 0, oldFormat);
                } else {
                    delete updateObject.format[insertIndex][action];
                    updateObject.format.splice(insertIndex, 0, oldFormat);
                }
            } else {
                // Format object needs to be split at start and end
                let oldFormatStart = { ...updateObject.format[insertIndex] };
                oldFormatStart.end = updateObject.format[insertIndex].start = totalOffset;
                let oldFormatEnd = { ...updateObject.format[insertIndex] };
                oldFormatEnd.start = updateObject.format[insertIndex].end = totalOffset + length;

                if (actionValue) {
                    updateObject.format[insertIndex][action] = attribute ?? actionValue;
                    updateObject.format.splice(insertIndex, 0, oldFormatStart);
                    updateObject.format.splice(insertIndex + 2, 0, oldFormatEnd);
                } else {
                    delete updateObject.format[insertIndex][action];
                    updateObject.format.splice(insertIndex, 0, oldFormatStart);
                    updateObject.format.splice(insertIndex + 2, 0, oldFormatEnd);
                }
            }
        } else {
            // Selection spans multiple elements

            if (updateObject.format[insertIndex]?.start > totalOffset) {
                // Selection starts outside the format object
                actionValue = true;

                updateObject.format.splice(insertIndex, 0, {
                    start: totalOffset,
                    end: updateObject.format[insertIndex].start,
                    [action]: true,
                });
            } else {
                // Selection starts inside the format object
                if (updateObject.format[insertIndex]?.start === totalOffset) {
                    // Format object is not split at start
                    if (actionValue) {
                        updateObject.format[insertIndex][action] = attribute ?? actionValue;
                    } else {
                        delete updateObject.format[insertIndex][action];
                    }
                } else {
                    // Format object needs to be split at start
                    let oldFormat = { ...updateObject.format[insertIndex] };
                    oldFormat.end = updateObject.format[insertIndex].start = totalOffset;

                    if (actionValue) {
                        updateObject.format[insertIndex][action] = attribute ?? actionValue;
                        updateObject.format.splice(insertIndex, 0, oldFormat);
                    } else {
                        delete updateObject.format[insertIndex][action];
                        updateObject.format.splice(insertIndex, 0, oldFormat);
                    }
                }
            }
            insertIndex++;
            while (updateObject.format[insertIndex]?.end <= totalOffset + length) {
                if (actionValue) {
                    if (updateObject.format[insertIndex].start > updateObject.format[insertIndex - 1].end) {
                        updateObject.format.splice(insertIndex, 0, {
                            start: updateObject.format[insertIndex - 1].end,
                            end: updateObject.format[insertIndex].start,
                            [action]: true,
                        });
                    } else {
                        updateObject.format[insertIndex][action] = attribute ?? actionValue;
                    }
                } else {
                    delete updateObject.format[insertIndex][action];
                }
                insertIndex++;
            }

            if (insertIndex >= updateObject.format.length || updateObject.format[insertIndex]?.start >= totalOffset + length) {
                // Selection ends outside the format object
                if (actionValue) {
                    updateObject.format.splice(insertIndex, 0, {
                        start: updateObject.format[insertIndex - 1].end,
                        end: totalOffset + length,
                        [action]: true,
                    });
                }
                return;
            }
            if (actionValue && updateObject.format[insertIndex]?.start <= totalOffset + length) {
                // Create new format object until last one
                if (updateObject.format[insertIndex].start > updateObject.format[insertIndex - 1].end) {
                    updateObject.format.splice(insertIndex, 0, {
                        start: updateObject.format[insertIndex - 1].end,
                        end: updateObject.format[insertIndex].start,
                        [action]: true,
                    });
                    insertIndex++;
                }
            }

            if (updateObject.format[insertIndex]?.end !== totalOffset + length) {
                // Format object needs to be split at end
                let oldFormat = { ...updateObject.format[insertIndex] };
                oldFormat.start = updateObject.format[insertIndex].end = totalOffset + length;

                if (actionValue) {
                    updateObject.format[insertIndex][action] = attribute ?? actionValue;
                    updateObject.format.splice(insertIndex + 1, 0, oldFormat);
                } else {
                    delete updateObject.format[insertIndex][action];
                    updateObject.format.splice(insertIndex, 0, oldFormat);
                }
            }
        }
    }

    function cleanObject() {
        if (!richSelection) return;
        let updateObject = richObject[richSelection.start.index];
        if (!updateObject.format) return;
        // Check if the formats are equal and can be merged
        for (let i = 0; i < updateObject.format.length; i++) {
            if (Object.keys(updateObject.format[i]).length === 2 || updateObject.format[i].start === updateObject.format[i].end) {
                updateObject.format.splice(i, 1);
                i--;
                continue;
            }
            if (i < updateObject.format.length - 1) {
                if (updateObject.format[i].end !== updateObject.format[i + 1].start) continue;
                let merge = true;
                for (let prop in updateObject.format[i]) {
                    if (prop === 'start' || prop === 'end') continue;
                    if (updateObject.format[i][prop] !== updateObject.format[i + 1][prop]) {
                        merge = false;
                        break;
                    }
                }
                if (merge) {
                    // Other way around
                    for (let prop in updateObject.format[i + 1]) {
                        if (prop === 'start' || prop === 'end') continue;
                        if (updateObject.format[i][prop] !== updateObject.format[i + 1][prop]) {
                            merge = false;
                            break;
                        }
                    }
                }
                if (merge) {
                    updateObject.format[i].end = updateObject.format[i + 1].end;
                    updateObject.format.splice(i + 1, 1);
                    i--;
                }
            }
        }
        if (updateObject.format.length === 0) {
            delete updateObject.format;
        }
    }

    /** @param {import('./types').RichObject} object */
    function objectToHtml(object) {
        let html = '';
        for (const node of object) {
            html += `<${node.type}>${nodeToHtml(node)}</${node.type}>`;
        }
        return html;
    }
    /** @param {any} node*/
    function nodeToHtml(node) {
        let text = node.text;
        if (!node.format) {
            return text;
        }
        let markers = [];
        for (const format of node.format) {
            let formatStartTag = '';
            let formatEndTag = '';
            for (const prop in format) {
                if (prop === 'start' || prop === 'end') continue;
                if (format[prop]) {
                    if (prop === 'a') {
                        formatStartTag += `<a href="${format[prop]}">`;
                        formatEndTag = `</a>` + formatEndTag;
                    } else {
                        formatStartTag += `<${prop}>`;
                    }
                    formatEndTag = `</${prop}>` + formatEndTag;
                }
            }
            markers.push({
                index: format.start,
                tag: formatStartTag,
            });
            markers.push({
                index: format.end,
                tag: formatEndTag,
            });
        }
        markers.sort((a, b) => a.index - b.index);
        let offset = 0;
        for (const marker of markers) {
            text = text.slice(0, marker.index + offset) + marker.tag + text.slice(marker.index + offset);
            offset += marker.tag.length;
        }
        return text;
    }

    /**
     * @param {Event & { inputType: string, currentTarget: EventTarget & HTMLDivElement; }} event
     */
    function updateText(event) {
        if (!richSelection) {
            return;
        }
        if (event.inputType === 'insertParagraph') {
            let updateObject = richObject[richSelection.start.index];
            let newObject = {
                type: updateObject.type,
                text: updateObject.text.slice(richSelection.totalOffset),
            };
            updateObject.text = updateObject.text.slice(0, richSelection.totalOffset);
            if (updateObject.format) {
                newObject.format = [];
                for (let i = 0; i < updateObject.format.length; i++) {
                    let format = updateObject.format[i];
                    let newFormat = { ...format };
                    if (format.start > updateObject.text.length) {
                        newFormat.start -= updateObject.text.length;
                        newFormat.end -= updateObject.text.length;
                        newObject.format.push(newFormat);
                        updateObject.format.splice(i, 1);
                        i--;
                    } else if (format.end > updateObject.text.length) {
                        newFormat.start = 0;
                        newFormat.end -= updateObject.text.length;
                        format.end = updateObject.text.length;
                        newObject.format.push(newFormat);
                    }
                }
            }
            richObject.splice(richSelection.start.index + 1, 0, newObject);
            richObject = richObject;
            return;
        }

        if (
            (event.inputType === 'deleteContentBackward' && richSelection?.totalOffset === 0) ||
            (event.inputType === 'deleteContentForward' && richSelection?.totalOffset === richObject[richSelection.start.index].text.length)
        ) {
            if (event.inputType === 'deleteContentForward') {
                richSelection.start.index++;
            }
            let updateObject = richObject[richSelection.start.index];
            let prevObject = richObject[richSelection.start.index - 1];
            if (updateObject.format) {
                if (!prevObject.format) {
                    prevObject.format = [];
                }
                for (const format of updateObject.format) {
                    format.start += prevObject.text.length;
                    format.end += prevObject.text.length;
                    prevObject.format.push(format);
                }
            }
            prevObject.text += updateObject.text;
            richObject.splice(richSelection.start.index, 1);
            richObject = richObject;
            return;
        }

        if (
            (event.inputType === 'deleteContentBackward' || event.inputType === 'deleteContentForward') &&
            richSelection.start.index !== richSelection.end.index &&
            richSelection.length > 0
        ) {
            let length = richSelection.length;
            length -= richObject[richSelection.start.index].text.length - richSelection.start.offset;
            richObject[richSelection.start.index].text.slice(0, richSelection.totalOffset);
            for (let i = 0; i < richObject[richSelection.start.index].format.length; i++) {
                let format = richObject[richSelection.start.index].format[i];
                if (format.start > richSelection.totalOffset) {
                    richObject[richSelection.start.index].format.splice(i, 1);
                    i--;
                }
            }

            for (let i = richSelection.end.index - 1; i > richSelection.start.index; i--) {
                length -= richObject[i].text.length;
                richObject.splice(i, 1);
            }

            richObject[richSelection.start.index].text += richObject[richSelection.end.index].text.slice(length);
            richObject.splice(richSelection.end.index, 1);
            // TODO: update format
            return;
        }

        let childs = event.target.children;

        let lengthChange = childs[richSelection?.start.index].innerText.length - richObject[richSelection?.start.index].text.length;
        richObject[richSelection?.start.index].text = childs[richSelection?.start.index].innerText;

        if (!richObject[richSelection.start.index].format) {
            return;
        }

        let insertIndex = 0;
        while (
            richObject[richSelection.start.index].format[insertIndex] &&
            richObject[richSelection.start.index].format[insertIndex]?.end <= richSelection.totalOffset
        ) {
            insertIndex++;
        }
        while (insertIndex < richObject[richSelection.start.index].format.length) {
            if (richObject[richSelection.start.index].format[insertIndex].start > richSelection.totalOffset) {
                richObject[richSelection.start.index].format[insertIndex].start += lengthChange;
            }
            richObject[richSelection.start.index].format[insertIndex].end += lengthChange;
            insertIndex++;
        }
    }

    let linkWidged = false;
    let linkText = '';
    let linkSelection = richSelection;

    onMount(() => {
        document.addEventListener('selectionchange', handleSelectionChange);
        liveObject = content;
        richObject = content;
    });
</script>

<div class="richeditor">
    <div class="toolbar">
        <button on:click={() => updateStyle('b')}><b>B</b></button>
        <button on:click={() => updateStyle('i')}><i>I</i></button>
        <button on:click={() => updateStyle('u')}><u>U</u></button>
        <select value="paragraph" on:change={(e) => updateNodeType(e.target.value)}>
            <option value="paragraph">Paragraph</option>
            <option value="h1">Heading 1</option>
            <option value="h2">Heading 2</option>
            <option value="h3">Heading 3</option>
            <option value="h4">Heading 4</option>
            <option value="h5">Heading 5</option>
            <option value="h6">Heading 6</option>
            <option value="blockquote">Blockquote</option>
            <option value="code">Code</option>
        </select>
        <button
            class="link"
            on:click={() => {
                linkSelection = richSelection;
                linkWidged = true;
            }}
        >
            <svg
                xmlns="http://www.w3.org/2000/svg"
                class="icon icon-tabler icon-tabler-link"
                width="15"
                height="15"
                viewBox="0 0 24 24"
                stroke-width="2"
                stroke="currentColor"
                fill="none"
                stroke-linecap="round"
                stroke-linejoin="round"
            >
                <path stroke="none" d="M0 0h24v24H0z" fill="none" />
                <path d="M9 15l6 -6" />
                <path d="M11 6l.463 -.536a5 5 0 0 1 7.071 7.072l-.534 .464" />
                <path d="M13 18l-.397 .534a5.068 5.068 0 0 1 -7.127 0a4.972 4.972 0 0 1 0 -7.071l.524 -.463" />
            </svg>
        </button>
        {#if linkWidged}
            <div class="link-widget">
                <input on:mousedown|stopPropagation type="text" bind:value={linkText} placeholder="Link" />
                <button
                    on:click={() => {
                        richSelection = linkSelection;
                        updateStyle('a', linkText);
                        linkWidged = false;
                    }}>Add</button
                >
            </div>
        {/if}
    </div>

    <div id="editor" bind:this={editor} contenteditable="true" on:input={updateText}>
        {@html $html}
    </div>
</div>

<style>
    .richeditor {
        width: 50%;
        margin: 0 auto;
        border: 1px solid #ccc;
        border-radius: 4px;
    }
    #editor {
        padding: 1rem;
        min-height: 200px;
    }

    .toolbar {
        display: flex;
        align-items: center;
        position: relative;
        border-bottom: 1px solid #ccc;
        padding: 1rem;
    }
    button {
        background: #fff;
        border: 1px solid #ccc;
        border-radius: 4px;
        cursor: pointer;
        margin-right: 1rem;
        padding: 4px 8px;
        transition: all 0.2s ease-in-out;
    }
    button:hover {
        background: #f5f5f5;
    }
    select {
        background: #fff;
        border: 1px solid #ccc;
        border-radius: 4px;
        color: #333;
        cursor: pointer;
        font-size: 14px;
        padding: 4px 24px 4px 8px;
        transition: all 0.2s ease-in-out;
        width: 120px;
        margin-right: 1rem;
    }
    select:focus {
        border-color: #0077cc;
        outline: none;
    }

    button.link {
        display: flex;
        align-items: center;
    }

    .link-widget {
        position: absolute;
        top: 100%;
        left: 50%;
        transform: translateX(-50%);
        display: flex;
        align-items: center;
        justify-content: center;
        background-color: #fff;
        border: 1px solid #ccc;
        padding: 10px;
    }

    .link-widget input {
        margin-right: 10px;
        border: 1px solid #ccc;
        border-radius: 4px;
        padding: 5px;
    }

    .link-widget button {
        background-color: #007bff;
        color: #fff;
        border: none;
        padding: 5px 10px;
        border-radius: 4px;
        cursor: pointer;
    }
</style>
