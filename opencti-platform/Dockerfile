FROM node:20-alpine AS base

RUN corepack enable

FROM base AS graphql-deps-builder

WORKDIR /opt/opencti-build/opencti-graphql
COPY opencti-graphql/package.json opencti-graphql/yarn.lock opencti-graphql/.yarnrc.yml ./
COPY opencti-graphql/.yarn ./.yarn
COPY opencti-graphql/patch ./patch
RUN set -ex; \
    apk add --no-cache git tini gcc g++ make musl-dev cargo python3 python3-dev postfix postfix-pcre py3-setuptools  \
    && npm install -g node-gyp \
    && yarn install --frozen-lockfile && yarn cache clean --all


FROM base AS graphql-builder

WORKDIR /opt/opencti-build/opencti-graphql
COPY opencti-graphql/package.json opencti-graphql/yarn.lock opencti-graphql/.yarnrc.yml ./
COPY opencti-graphql/.yarn ./.yarn
COPY opencti-graphql/patch ./patch
RUN set -ex; \
    apk add --no-cache git tini gcc g++ make musl-dev cargo python3 python3-dev postfix postfix-pcre  py3-setuptools  \
    && npm install -g node-gyp \
    && yarn install
COPY opencti-graphql /opt/opencti-build/opencti-graphql
RUN yarn build:prod


FROM base AS front-builder

WORKDIR /opt/opencti-build/opencti-front

COPY opencti-front/package.json opencti-front/yarn.lock opencti-front/.yarnrc.yml ./
COPY opencti-front/.yarn ./.yarn
COPY opencti-front/patch ./patch
COPY opencti-front/packages ./packages
RUN set -ex; \
    apk add --no-cache git tini gcc g++ make musl-dev cargo python3 python3-dev postfix postfix-pcre py3-setuptools  \
    && npm install -g node-gyp \
&& yarn install

COPY opencti-front /opt/opencti-build/opencti-front
COPY opencti-graphql/config/schema/opencti.graphql /opt/opencti-build/opencti-graphql/config/schema/opencti.graphql

RUN yarn build:standalone


FROM base AS app

WORKDIR /opt/opencti

RUN set -ex; \
    apk add --no-cache git tini gcc g++ make musl-dev cargo python3 python3-dev postfix postfix-pcre py3-pip py3-setuptools \
    && rm -f /usr/lib/python3.11/EXTERNALLY-MANAGED \
    && rm -f /usr/lib/python3.12/EXTERNALLY-MANAGED \
    && python3 -m venv venv \
    && source venv/bin/activate \
    && rm -rv /usr/lib/python*/ensurepip \
    && pip3 install --no-cache-dir --upgrade pip setuptools wheel \
    && ln -sf venv/bin/python3 /usr/bin/python


COPY opencti-graphql/src/python/requirements.txt ./src/python/requirements.txt

RUN cat ./src/python/requirements.txt

RUN source venv/bin/activate \
&& pip3 install --no-cache-dir --requirement ./src/python/requirements.txt \
    && pip3 install \
        --no-cache-dir \
        --upgrade \
        --force 'git+https://github.com/OpenCTI-Platform/client-python.git@5.11.13' 
#    && apk del git gcc musl-dev

COPY --from=graphql-deps-builder /opt/opencti-build/opencti-graphql/node_modules ./node_modules
COPY --from=graphql-builder /opt/opencti-build/opencti-graphql/build ./build
COPY --from=graphql-builder /opt/opencti-build/opencti-graphql/static ./static
COPY --from=front-builder /opt/opencti-build/opencti-front/builder/prod/build ./public
COPY opencti-graphql/src ./src
COPY opencti-graphql/config ./config
COPY opencti-graphql/script ./script

ARG UID=10000
ARG GID=10001
ARG USERNAME="_opencti"

ENV PYTHONUNBUFFERED=1
ENV NODE_OPTIONS=--max_old_space_size=12288
ENV NODE_ENV=production

RUN set -ex \
    ; addgroup -g "${GID}" -S "${USERNAME}" \
    && adduser \
        -h /opt/opencti \
        -g "OpenCTI privsep user" \
        -s "/sbin/nologin" \
        -G "${USERNAME}" \
        -S \
        -u "${UID}" \
        "${USERNAME}" \
    && install -o "${UID}" -g "${GID}" -m 0750 -d '/opt/opencti/logs'

VOLUME ["/opt/opencti/logs"]

USER "${USERNAME}"

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "build/back.js"]