servicenow-client-scripts

Client Scripts, Script Includes, and UI Policies I built while learning ServiceNow development. All scripts were written and tested live in my Personal Developer Instance (PDI).

What's in here
1. onChange — Urgency Warning (critical-urgency-warning.js)
Shows a colored message under the Urgency field whenever the value changes on an Incident form. Different color for each urgency level.

Urgency 1 → red error message
Urgency 2 → yellow warning message
Urgency 3 → blue info message
Anything else → message disappears
Settings: Table: Incident | Type: onChange | Field: Urgency

function onChange(control, oldValue, newValue, isLoading) {
    if (isLoading) {
        return;
    }

    if (newValue == '1') {
        g_form.showFieldMsg('urgency', 'Critical urgency — assign immediately!', 'error');
    } else if (newValue == '2') {
        g_form.showFieldMsg('urgency', 'High urgency — assign within 2 hours', 'warning');
    } else if (newValue == '3') {
        g_form.showFieldMsg('urgency', 'Low urgency — assign when available', 'info');
    } else {
        g_form.hideFieldMsg('urgency');
    }
}
2. onLoad — Urgency Warning on Form Open (urgency-warning-onload.js)
Same warning as above but fires when the form first opens. Without this, if someone opens an incident that already has Urgency = 1, no message shows because the field never "changed". This fills that gap.

Settings: Table: Incident | Type: onLoad

function onLoad() {
    var urgency = g_form.getValue('urgency');

    if (urgency == '1') {
        g_form.showFieldMsg('urgency', 'Critical urgency — assign immediately!', 'error');
    } else if (urgency == '2') {
        g_form.showFieldMsg('urgency', 'High urgency — assign within 2 hours', 'warning');
    } else if (urgency == '3') {
        g_form.showFieldMsg('urgency', 'Low urgency — assign when available', 'info');
    }
}
3. onSubmit — Short Description Validation (validate-short-description.js)
Blocks the incident from saving if the short description is too short or doesn't contain at least one IT-related keyword. Prevents vague submissions like "issue" or "help".

Settings: Table: Incident | Type: onSubmit

function onSubmit() {
    var shortDesc = g_form.getValue('short_description').toLowerCase().trim();

    if (shortDesc.length < 10) {
        g_form.showFieldMsg('short_description',
            'Short description must be at least 10 characters', 'error');
        return false;
    }

    var relevantWords = ['error', 'access', 'password', 'login', 'network',
                         'slow', 'crash', 'install', 'broken', 'failed',
                         'cannot', 'unable', 'request', 'reset', 'email',
                         'printer', 'screen', 'laptop', 'system', 'server'];

    var hasRelevantWord = false;
    for (var i = 0; i < relevantWords.length; i++) {
        if (shortDesc.indexOf(relevantWords[i]) !== -1) {
            hasRelevantWord = true;
            break;
        }
    }

    if (!hasRelevantWord) {
        g_form.showFieldMsg('short_description',
            'Please describe the actual IT issue (e.g. cannot login, printer broken)', 'error');
        return false;
    }
}
Note: Keyword matching is the current approach. A future version will call an AI API via RESTMessageV2 to check semantic relevance — whether the sentence actually describes an IT problem, not just whether it contains a known word.

4. onChange — Conditional Mandatory Field (assigned-to-mandatory.js)
Makes the Assigned To field mandatory when Impact is set to 1 - High. Removes the requirement when Impact changes to anything else. This is a common pattern for conditional validation that the dictionary alone can't handle.

Settings: Table: Incident | Type: onChange | Field: Impact

function onChange(control, oldValue, newValue, isLoading) {
    if (isLoading) {
        return;
    }

    if (newValue == '1') {
        g_form.setMandatory('assigned_to', true);
        g_form.showFieldMsg('assigned_to',
            'High impact — must be assigned before saving', 'error');
    } else {
        g_form.setMandatory('assigned_to', false);
        g_form.hideFieldMsg('assigned_to');
    }
}
5. Script Include — IncidentUtils (IncidentUtils.js)
A reusable server-side utility class for incident-related logic. Instead of writing the same code in multiple Business Rules, this class gets called from anywhere using new IncidentUtils().

Two methods:

isOverdue(incidentNumber) — returns true if the incident has been open more than 24 hours
getPriorityLabel(priorityValue) — returns a human-readable priority description
Settings: Name: IncidentUtils | Accessible from: All application scopes | Active: true

var IncidentUtils = Class.create();
IncidentUtils.prototype = {
    initialize: function() {},

    isOverdue: function(incidentNumber) {
        var gr = new GlideRecord('incident');
        gr.addQuery('number', incidentNumber);
        gr.query();

        if (gr.next()) {
            var opened = gr.getValue('opened_at');
            var openedDate = new GlideDateTime(opened);
            var now = new GlideDateTime();
            var diff = GlideDateTime.subtract(openedDate, now);
            var hours = diff.getRoundedDayPart() * 24;

            if (hours >= 24) {
                return true;
            }
        }
        return false;
    },

    getPriorityLabel: function(priorityValue) {
        var labels = {
            '1': 'Critical - Immediate action required',
            '2': 'High - Respond within 2 hours',
            '3': 'Moderate - Respond within 8 hours',
            '4': 'Low - Respond within 24 hours',
            '5': 'Planning - No SLA applied'
        };
        return labels[priorityValue] || 'Unknown priority';
    },

    type: 'IncidentUtils'
};
Tested in Scripts - Background:

var utils = new IncidentUtils();
gs.info(utils.getPriorityLabel('1'));  // Critical - Immediate action required
gs.info(utils.getPriorityLabel('3'));  // Moderate - Respond within 8 hours
gs.info(utils.isOverdue('INC0010005')); // true
6. UI Policy — Network Category Makes CI Mandatory
No-code field control. When Category is set to Network on an Incident, the Configuration Item field becomes mandatory and visible automatically. No JavaScript needed.

Settings:

Table: Incident
Condition: Category is Network
Active: true | On load: true | Global: true
UI Policy Action:

Field: Configuration item
Mandatory: True
Visible: True
Read only: Ignore
Proof it works
urgency warning showing on incident form

Stack
ServiceNow PDI
Client Scripts (onLoad, onChange, onSubmit)
Script Includes (Class.create pattern)
UI Policies
GlideRecord, GlideDateTime
g_form API
What's next
Future version of the short description validator will call an AI API via RESTMessageV2 to check if the sentence semantically describes an IT problem — not just keyword matching. The outbound REST infrastructure for this is already built from Week 1.# servicenow-client-scripts
