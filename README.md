

#include "health.h"

#define HEALTH_CONF_MAX_LINE 4096

#define HEALTH_ALARM_KEY "alarm"
#define HEALTH_TEMPLATE_KEY "template"
#define HEALTH_ON_KEY "on"
#define HEALTH_HOST_KEY "hosts"
#define HEALTH_OS_KEY "os"
#define HEALTH_FAMILIES_KEY "families"
#define HEALTH_PLUGIN_KEY "plugin"
#define HEALTH_MODULE_KEY "module"
#define HEALTH_CHARTS_KEY "charts"
#define HEALTH_LOOKUP_KEY "lookup"
#define HEALTH_CALC_KEY "calc"
#define HEALTH_EVERY_KEY "every"
#define HEALTH_GREEN_KEY "green"
#define HEALTH_RED_KEY "red"
#define HEALTH_WARN_KEY "warn"
#define HEALTH_CRIT_KEY "crit"
#define HEALTH_EXEC_KEY "exec"
#define HEALTH_RECIPIENT_KEY "to"
#define HEALTH_UNITS_KEY "units"
#define HEALTH_INFO_KEY "info"
#define HEALTH_CLASS_KEY "class"
#define HEALTH_COMPONENT_KEY "component"
#define HEALTH_TYPE_KEY "type"
#define HEALTH_DELAY_KEY "delay"
#define HEALTH_OPTIONS_KEY "options"
#define HEALTH_REPEAT_KEY "repeat"
#define HEALTH_HOST_LABEL_KEY "host labels"
#define HEALTH_FOREACH_KEY "foreach"
#define HEALTH_CHART_LABEL_KEY "chart labels"

static inline int health_parse_delay(
        size_t line, const char *filename, char *string,
        int *delay_up_duration,
        int *delay_down_duration,
        int *delay_max_duration,
        float *delay_multiplier) {

    char given_up = 0;
    char given_down = 0;
    char given_max = 0;
    char given_multiplier = 0;

    char *s = string;
    while(*s) {
        char *key = s;

        while(*s && !isspace(*s)) s++;
        while(*s && isspace(*s)) *s++ = '\0';

        if(!*key) break;

        char *value = s;
        while(*s && !isspace(*s)) s++;
        while(*s && isspace(*s)) *s++ = '\0';

        if(!strcasecmp(key, "up")) {
            if (!config_parse_duration(value, delay_up_duration)) {
                error("Health configuration at line %zu of file '%s': invalid value '%s' for '%s' keyword",
                        line, filename, value, key);
            }
            else given_up = 1;
        }
        else if(!strcasecmp(key, "down")) {
            if (!config_parse_duration(value, delay_down_duration)) {
                error("Health configuration at line %zu of file '%s': invalid value '%s' for '%s' keyword",
                        line, filename, value, key);
            }
            else given_down = 1;
        }
        else if(!strcasecmp(key, "multiplier")) {
            *delay_multiplier = strtof(value, NULL);
            if(isnan(*delay_multiplier) || isinf(*delay_multiplier) || islessequal(*delay_multiplier, 0)) {
                error("Health configuration at line %zu of file '%s': invalid value '%s' for '%s' keyword",
                        line, filename, value, key);
            }
            else given_multiplier = 1;
        }
        else if(!strcasecmp(key, "max")) {
            if (!config_parse_duration(value, delay_max_duration)) {
                error("Health configuration at line %zu of file '%s': invalid value '%s' for '%s' keyword",
                        line, filename, value, key);
            }
            else given_max = 1;
        }
        else {
            error("Health configuration at line %zu of file '%s': unknown keyword '%s'",
                    line, filename, key);
        }
    }

    if(!given_up)
        *delay_up_duration = 0;

    if(!given_down)
        *delay_down_duration = 0;

    if(!given_multiplier)
        *delay_multiplier = 1.0;

    if(!given_max) {
        if((*delay_max_duration) < (*delay_up_duration) * (*delay_multiplier))
            *delay_max_duration = (int)((*delay_up_duration) * (*delay_multiplier));

        if((*delay_max_duration) < (*delay_down_duration) * (*delay_multiplier))
            *delay_max_duration = (int)((*delay_down_duration) * (*delay_multiplier));
    }

    return 1;
}

static inline uint32_t health_parse_options(const char *s) {
    uint32_t options = 0;
    char buf[100+1] = "";

    while(*s) {
        buf[0] = '\0';

        // skip spaces
        while(*s && isspace(*s))
            s++;

        // find the next space
