# esphome-ha-github-shell-dashboard
Configuration to perform simple Git pull / push commands from dashboard buttons to github

# Home Assistant Git Control Dashboard

## Step 1: Create Scripts in configuration.yaml

Add these scripts to your `configuration.yaml`:

```yaml
script:
  esphome_git_pull:
    alias: "ESPHome Git Pull"
    sequence:
      - service: shell_command.esphome_pull_with_output
      - delay:
          seconds: 2
      - service: homeassistant.update_entity
        data:
          entity_id: 
            - sensor.esphome_git_pull_output
            - sensor.esphome_git_pull_error

  esphome_git_commit:
    alias: "ESPHome Git Commit"
    sequence:
      - service: shell_command.esphome_commit_with_output
      - delay:
          seconds: 2
      - service: homeassistant.update_entity
        data:
          entity_id: 
            - sensor.esphome_git_commit_output
            - sensor.esphome_git_commit_error

  esphome_git_push:
    alias: "ESPHome Git Push"
    sequence:
      - service: shell_command.esphome_push_with_output
      - delay:
          seconds: 2
      - service: homeassistant.update_entity
        data:
          entity_id: 
            - sensor.esphome_git_push_output
            - sensor.esphome_git_push_error
```

## Step 2: Update Shell Commands

Modify your shell commands to capture output:

```yaml
shell_command:
  esphome_pull_with_output: >
    cd /config/esphome && GIT_SSH_COMMAND='ssh -i /.ssh/github_key -o IdentitiesOnly=yes -o StrictHostKeyChecking=no' git pull > /config/esphome_git_pull_output.txt 2> /config/esphome_git_pull_error.txt
  
  esphome_commit_with_output: >
    cd /config/esphome && git add . && git commit -a -m "esphome dashboard edit" > /config/esphome_git_commit_output.txt 2> /config/esphome_git_commit_error.txt
  
  esphome_push_with_output: >
    cd /config/esphome && GIT_SSH_COMMAND='ssh -i /.ssh/github_key -o IdentitiesOnly=yes -o StrictHostKeyChecking=no' git push > /config/esphome_git_push_output.txt 2> /config/esphome_git_push_error.txt
```

Note: The commit command now includes `git add .` to ensure all changes are staged before committing.

## Step 3: Create Command Line Sensors

Add these sensors to read the output files and timestamps:

```yaml
command_line:
  - sensor:
      name: ESPHome Git Pull Output
      command: "cat /config/esphome_git_pull_output.txt 2>/dev/null || echo 'No output'"
      scan_interval: 5
      command_timeout: 5
      value_template: "{{ value[:255] }}"
      
  - sensor:
      name: ESPHome Git Pull Error
      command: "cat /config/esphome_git_pull_error.txt 2>/dev/null || echo 'No errors'"
      scan_interval: 5
      command_timeout: 5
      value_template: "{{ value[:255] }}"
      
  - sensor:
      name: ESPHome Git Pull Timestamp
      command: "stat -c %Y /config/esphome_git_pull_output.txt 2>/dev/null || echo '0'"
      scan_interval: 5
      command_timeout: 5
      value_template: >
        {% set timestamp = value | int %}
        {% if timestamp > 0 %}
          {{ timestamp | timestamp_custom('%Y-%m-%d %H:%M:%S') }}
        {% else %}
          Never
        {% endif %}
      
  - sensor:
      name: ESPHome Git Commit Output
      command: "cat /config/esphome_git_commit_output.txt 2>/dev/null || echo 'No output'"
      scan_interval: 5
      command_timeout: 5
      value_template: "{{ value[:255] }}"
      
  - sensor:
      name: ESPHome Git Commit Error
      command: "cat /config/esphome_git_commit_error.txt 2>/dev/null || echo 'No errors'"
      scan_interval: 5
      command_timeout: 5
      value_template: "{{ value[:255] }}"
      
  - sensor:
      name: ESPHome Git Commit Timestamp
      command: "stat -c %Y /config/esphome_git_commit_output.txt 2>/dev/null || echo '0'"
      scan_interval: 5
      command_timeout: 5
      value_template: >
        {% set timestamp = value | int %}
        {% if timestamp > 0 %}
          {{ timestamp | timestamp_custom('%Y-%m-%d %H:%M:%S') }}
        {% else %}
          Never
        {% endif %}
      
  - sensor:
      name: ESPHome Git Push Output
      command: "cat /config/esphome_git_push_output.txt 2>/dev/null || echo 'No output'"
      scan_interval: 5
      command_timeout: 5
      value_template: "{{ value[:255] }}"
      
  - sensor:
      name: ESPHome Git Push Error
      command: "cat /config/esphome_git_push_error.txt 2>/dev/null || echo 'No errors'"
      scan_interval: 5
      command_timeout: 5
      value_template: "{{ value[:255] }}"
      
  - sensor:
      name: ESPHome Git Push Timestamp
      command: "stat -c %Y /config/esphome_git_push_output.txt 2>/dev/null || echo '0'"
      scan_interval: 5
      command_timeout: 5
      value_template: >
        {% set timestamp = value | int %}
        {% if timestamp > 0 %}
          {{ timestamp | timestamp_custom('%Y-%m-%d %H:%M:%S') }}
        {% else %}
          Never
        {% endif %}
```

### Alternative: Using Template Sensors with Relative Time

If you prefer relative timestamps (e.g., "5 minutes ago"), add these template sensors:

```yaml
template:
  - sensor:
      - name: "ESPHome Git Pull Last Run"
        state: >
          {% set timestamp = states('sensor.esphome_git_pull_timestamp') %}
          {% if timestamp not in ['unknown', 'unavailable', 'Never'] %}
            {% set last_run = strptime(timestamp, '%Y-%m-%d %H:%M:%S') %}
            {% set time_diff = now() - last_run %}
            {% set minutes = (time_diff.total_seconds() / 60) | int %}
            {% set hours = (minutes / 60) | int %}
            {% set days = (hours / 24) | int %}
            {% if days > 0 %}
              {{ days }} day{{ 's' if days > 1 else '' }} ago
            {% elif hours > 0 %}
              {{ hours }} hour{{ 's' if hours > 1 else '' }} ago
            {% elif minutes > 0 %}
              {{ minutes }} minute{{ 's' if minutes > 1 else '' }} ago
            {% else %}
              Just now
            {% endif %}
          {% else %}
            Never
          {% endif %}
        icon: mdi:clock-outline
```

Then use `{{ states('sensor.esphome_git_pull_last_run') }}` in your dashboard for a more user-friendly display.

## Step 4: Create Lovelace Dashboard

Add this card to your Lovelace dashboard:

```yaml
type: vertical-stack
cards:
  - type: markdown
    content: |
      # ESPHome Git Control
      
  - type: horizontal-stack
    cards:
      - show_name: true
        show_icon: true
        type: button
        name: Git Pull
        icon: mdi:download
        tap_action:
          action: call-service
          service: script.esphome_git_pull
        entity: script.esphome_git_pull
        
      - show_name: true
        show_icon: true
        type: button
        name: Git Commit
        icon: mdi:check
        tap_action:
          action: call-service
          service: script.esphome_git_commit
        entity: script.esphome_git_commit
        
      - show_name: true
        show_icon: true
        type: button
        name: Git Push
        icon: mdi:upload
        tap_action:
          action: call-service
          service: script.esphome_git_push
        entity: script.esphome_git_push
        
  - type: markdown
    content: |-
      ## Git Pull Output
      *Last run: {{ states('sensor.esphome_git_pull_timestamp') }}*
      **STDOUT:** {{ states('sensor.esphome_git_pull_output') }}
      **STDERR:** {{ states('sensor.esphome_git_pull_error') }}
        
  - type: markdown
    content: |-
      ## Git Commit Output
      *Last run: {{ states('sensor.esphome_git_commit_timestamp') }}*
      **STDOUT:** {{ states('sensor.esphome_git_commit_output') }}
      **STDERR:** {{ states('sensor.esphome_git_commit_error') }}
        
  - type: markdown
    content: |-
      ## Git Push Output
      *Last run: {{ states('sensor.esphome_git_push_timestamp') }}*
      **STDOUT:** {{ states('sensor.esphome_git_push_output') }}
      **STDERR:** {{ states('sensor.esphome_git_push_error') }}
```
## Notes:

1. **Restart Required**: After adding the configuration changes, restart Home Assistant.

2. **Permissions**: Ensure the Home Assistant user has write permissions to `/config/` directory.

3. **Output Limitations**: Command line sensors have a 255 character limit. For longer outputs, consider using a custom script that truncates output or stores it in chunks.

4. **Security**: Be careful with git operations that might expose sensitive information in the output.
        
