# Megaphone — strip Meta-deprecated scopes Postiz still hardcodes, persistently.
# These scopes are invalid for ALL apps now and break Meta OAuth until removed.
#   read_insights            (Postiz #1346)
#   instagram_basic          (Postiz #1578)
#   instagram_manage_insights
FROM ghcr.io/gitroomhq/postiz-app:latest
RUN for f in $(grep -rEl 'read_insights|instagram_basic|instagram_manage_insights' /app); do \
      sed -i -E "s/['\"]read_insights['\"],?//g; \
                 s/['\"]instagram_basic['\"],?//g; \
                 s/['\"]instagram_manage_insights['\"],?//g" "$f"; \
    done
