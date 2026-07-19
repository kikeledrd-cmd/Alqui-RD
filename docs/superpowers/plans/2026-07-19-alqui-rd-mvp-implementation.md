# Alqui-RD MVP — Plan de implementación

Este documento funciona como índice maestro del plan técnico aprobado para construir Alqui-RD.

El plan completo está dividido en módulos para facilitar su lectura, revisión y ejecución por agentes de desarrollo sin perder el orden original de las 19 tareas.

## Objetivo

Construir una PWA inmobiliaria dominicana donde agentes aprobados publiquen propiedades, visitantes las encuentren y contacten, y Alqui-RD registre y gestione cada oportunidad hasta la visita y el cierre.

## Arquitectura

Monolito modular con portal público, consola profesional y consola administrativa. Supabase proporciona PostgreSQL, autenticación y almacenamiento; las escrituras se realizan mediante acciones de servidor o rutas protegidas, con permisos por rol, RLS y auditoría.

## Stack principal

- Node.js 22
- pnpm 10
- Next.js 15
- React 19
- TypeScript 5
- Tailwind CSS 4
- Supabase PostgreSQL, Auth y Storage
- Zod y React Hook Form
- Vitest y Testing Library
- Playwright
- MapLibre GL
- Adaptador de correo compatible con Resend
- Rutas cron compatibles con Vercel

## Módulos del plan

1. [Visión general, restricciones, estructura e interfaces](./2026-07-19-alqui-rd-mvp-implementation/00-overview.md)
2. [Ola 1 — Foundation: tareas 1 a 4](./2026-07-19-alqui-rd-mvp-implementation/01-foundation.md)
3. [Ola 2 — Supply Engine: tareas 5 a 10](./2026-07-19-alqui-rd-mvp-implementation/02-supply-engine.md)
4. [Ola 3 — Conversion Engine: tareas 11 a 13](./2026-07-19-alqui-rd-mvp-implementation/03-conversion-engine.md)
5. [Ola 4 — Business Engine: tareas 14 a 17](./2026-07-19-alqui-rd-mvp-implementation/04-business-engine.md)
6. [Ola 5 — Hardening & Launch: tareas 18 a 19](./2026-07-19-alqui-rd-mvp-implementation/05-hardening-launch.md)

## Orden obligatorio de ejecución

1. Foundation — tareas 1–4.
2. Supply Engine — tareas 5–10.
3. Conversion Engine — tareas 11–13.
4. Business Engine — tareas 14–17.
5. Hardening and Launch — tareas 18–19.

Cada tarea debe desarrollarse con TDD, terminar con pruebas verificables y producir un commit independiente antes de avanzar.

## Documento de diseño relacionado

- [Diseño funcional y técnico aprobado](../specs/2026-07-19-alqui-rd-mvp-design.md)
