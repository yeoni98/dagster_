
# Dagster 

이 문서는 Dagster의 핵심 개념과 케이크 제작 파이프라인 예제를 통해 실제 사용법을 설명합니다.

## 목차
- [Data Orchestrator란](#data-orchestrator란)
- [핵심 구성 요소](#핵심-구성-요소)
  - [Op](#op)
  - [Graph](#graph)
  - [Job](#job)
  - [Asset](#asset)
- [다른 asset에 dependency 주입 방법](#다른-asset에-dependency-주입-방법)
  - [Basic dependency](#basic-dependency)
  - [Different code locations](#different-code-locations)
- [자산 간 데이터 전달 방법](#자산-간-데이터-전달-방법)
  - [외부 저장소를 통한 명시적 데이터 관리](#1-외부-저장소를-통한-명시적-데이터-관리)
  - [IO 매니저를 사용한 암시적 데이터 관리](#2-io-매니저를-사용한-암시적-데이터-관리)
  - [자산 통합을 통한 데이터 전달 회피](#3-자산-통합을-통한-데이터-전달-회피)
- [메타데이터 관리](#메타데이터-관리)
  - [메타데이터 및 태그](#메타데이터-및-태그)
  - [Output vs add_output_metadata vs MaterializeResult](#output-vs-add_output_metadata-vs-materializeresult)
- [자동화](#자동화)
  - [스케줄 (Schedule)](#스케줄-schedule)
  - [센서 (Sensor)](#센서-sensor)
- [코드 로케이션](#코드-로케이션)
- [artifact](#아티팩트-관리)
- [예제](#예제)
