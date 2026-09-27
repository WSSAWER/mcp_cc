# Опубликованные инструменты 1C Database MCP

Источник: реестр `tools/list`. Всего инструментов: 53.

## Проекты и состояние

Настройка aliases ИБ, проверка привязок и просмотр экземпляра.

- `database_binding_list`
- `database_binding_get`
- `database_instance_get`
- `database_topology_get`
- `database_binding_actions_get`
- `database_binding_save`
- `database_binding_remove`
- `database_binding_readiness_check`

## Подключения

Логические lease для Designer, ibcmd и ИБ.

- `connection_lease_open`
- `connection_lease_get`
- `connection_lease_close`

## Конфигурация и расширения

Инвентаризация компонентов и настройка их Git/хранилища.

- `component_git_source_inspect`
- `component_list`
- `component_source_configure`
- `git_ssh_identity_prepare`
- `git_ssh_checkout_connect`

## Синхронизация

Разовые и фоновые задания, экспорт в Git и история.

- `sync_once_start`
- `sync_git_export_start`
- `sync_auto_enable`
- `sync_auto_resume`
- `sync_job_status_get`
- `sync_git_import_history_get`
- `sync_job_disable`

## Хранилище 1С

Получение, захват, освобождение и фиксация объектов.

- `repository_configuration_update`
- `repository_object_lock`
- `repository_object_unlock`
- `repository_object_commit`
- `repository_command_preview`
- `repository_command_run`
- `repository_object_revised_get`

## XML объекты

Планирование выгрузки, выгрузка, изменение и добавление объектов.

- `infobase_xml_export_preview`
- `infobase_xml_export_start`
- `infobase_xml_existing_files_update`
- `infobase_xml_new_objects_add`
- `infobase_xml_existing_roots_update`

## CF и CFE

Полная выгрузка и загрузка конфигурации или расширения.

- `infobase_binary_configuration_export`
- `infobase_binary_configuration_replace`

## ibcmd и расширения

Команды ibcmd, установка расширения и его флаги.

- `ibcmd_command_preview`
- `ibcmd_command_run`
- `extension_binary_load_preview`
- `extension_binary_load_run`
- `extension_flags_update_preview`
- `extension_flags_update_run`

## RAC и сеансы

Кластеры, информационные базы и пользовательские сеансы.

- `rac_cluster_list`
- `rac_infobase_list`
- `rac_session_list`
- `rac_session_terminate`

## Операции и логи

Очередь, отмена, результаты и поиск по логам.

- `operation_list`
- `log_tail_get`
- `log_command_list`
- `log_search`
- `operation_terminate`
- `operation_test_wait`
