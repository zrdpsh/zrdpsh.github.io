---
title: "Цикломатическая сложность"
date: 2024-09-14
---

Недавно в работе встретилось понятие цикломатической сложности.
Описывать, даже коротко, лучше с живым примером (ниже) - у нас будет единственный метод, который, имея на входе requestCode, попробует получить для приложения разрешение у Андроида пользоваться камерой, геолокацией, контактами и т.д.

Цикломатическая сложность - это "quantitative measure of the number of linearly independent paths through a program's source code", 

**Что за метрика?**
Идея: код сам по себе - это линейное (сверху вниз, строчка за строчкой) представление нелинейной, по определению, логики программы. Базово, функциональность обеспечивает комбинация из условий if  и циклов - выбрать, что сделать в зависимости от полученных данных, и повторить выбранное действие сколь угодно много раз. Наша порция данных, полученная на входе, движется через функцию и пройдёт одним из путей. Чем больше условий - тем больше маршрутов. Каждое условие добавляет минимум один новый вариант исполнения программы - с ним и без него. В качестве условий считаются, естественно, if-else, else, while, catch, switch, логические и тернарные операторы - всё, что варьирует поведение метода.
.. и по сути, мы уже мыслим метод в качестве направленного графа. Соотвественно, можно пересчитать узлы, они же точки принятия решений (Nodes), пересчитать рёбра между ними (Edges), принять количество "связанных компонентов" равным единице (exit Points, станет нужен, когда функций будет несколько), и подставить всё это в (Edges − Nodes + 2*Points). В нашем случае ЦС получается равной 22.
Ещё проще вариант - взять только точки принятия решений и прибавить единицу.

**В чём польза всего этого?**
Второе из двух важных свойств ЦС - кроме сложности, она же обозначает наименьшее количество юнит-тестов, которые требуются для нашего метода. Один путь через программу - плюс минимум один тест-кейс.
А первое важное свойство - мы подходим к коду формально, вообще не касаясь того, что делает программа и сводим всё объявленные сущности к одному-единственному, чаще всего двузначному, числу - и оно скажет, насколько легко (или не очень) будет читать и редактировать свой же собственную программу через пару дней. 
Код, лесенкой уходящий за правый край экрана, напрягал всегда, но теперь у интуиции есть название и точное значение. Наша checkPermissionsResult - далеко не самый суровый вариант, который можно встретить, и ЦС, равная 22 - считается вполне приемлемой. Но можно лучше.


**Насколько лучше?**
Чтобы было меньше 10 и как можно ближе к единице.
Удобство в том, что ЦС определяется сугубо формально - и снижается она тоже формальным методами. В теории, и рефакторить можно, не вникая, или почти не вникая, в суть того, что делает программа. Методов снижения ЦС несколько - в первую очередь, давайте попробуем самый "неинвазивный", и пока не будем делить функцию на части.

```
    protected boolean checkPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        if (grantResults == null) {
            grantResults = new int[0];
        }
        if (permissions == null) {
            permissions = new String[0];
        }

        boolean granted = grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED;

        if (requestCode == 104) {
            if (granted) {
                if (GroupCallActivity.groupCallInstance != null) {
                    GroupCallActivity.groupCallInstance.enableCamera();
                }

            } else {
                showPermissionErrorAlert(R.raw.permission_request_camera, LocaleController.getString(R.string.VoipNeedCameraPermission));
            }
        } else if (requestCode == REQUEST_CODE_EXTERNAL_STORAGE || requestCode == REQUEST_CODE_EXTERNAL_STORAGE_FOR_AVATAR) {
            if (!granted) {
                showPermissionErrorAlert(R.raw.permission_request_folder, requestCode == REQUEST_CODE_EXTERNAL_STORAGE_FOR_AVATAR ? LocaleController.getString(R.string.PermissionNoStorageAvatar) :
                        LocaleController.getString(R.string.PermissionStorageWithHint));
            } else {
                ImageLoader.getInstance().checkMediaPaths();
            }
        } else if (requestCode == REQUEST_CODE_ATTACH_CONTACT) {
            if (!granted) {
                showPermissionErrorAlert(R.raw.permission_request_contacts, LocaleController.getString(R.string.PermissionNoContactsSharing));
                return false;
            } else {
                ContactsController.getInstance(currentAccount).forceImportContacts();
            }
        } else if (requestCode == 3 || requestCode == REQUEST_CODE_VIDEO_MESSAGE) {
            boolean audioGranted = true;
            boolean cameraGranted = true;
            for (int i = 0, size = Math.min(permissions.length, grantResults.length); i < size; i++) {
                if (Manifest.permission.RECORD_AUDIO.equals(permissions[i])) {
                    audioGranted = grantResults[i] == PackageManager.PERMISSION_GRANTED;
                } else if (Manifest.permission.CAMERA.equals(permissions[i])) {
                    cameraGranted = grantResults[i] == PackageManager.PERMISSION_GRANTED;
                }
            }
            if (requestCode == REQUEST_CODE_VIDEO_MESSAGE && (!audioGranted || !cameraGranted)) {
                showPermissionErrorAlert(R.raw.permission_request_camera, LocaleController.getString(R.string.PermissionNoCameraMicVideo));
            } else if (!audioGranted) {
                showPermissionErrorAlert(R.raw.permission_request_microphone, LocaleController.getString(R.string.PermissionNoAudioWithHint));
            } else if (!cameraGranted) {
                showPermissionErrorAlert(R.raw.permission_request_camera, LocaleController.getString(R.string.PermissionNoCameraWithHint));
            } else {
                if (SharedConfig.inappCamera) {
                    CameraController.getInstance().initCamera(null);
                }
                return false;
            }
        } else if (requestCode == 18 || requestCode == 19 || requestCode == REQUEST_CODE_OPEN_CAMERA || requestCode == 22) {
            if (!granted) {
                showPermissionErrorAlert(R.raw.permission_request_camera, LocaleController.getString(R.string.PermissionNoCameraWithHint));
            }
        } else if (requestCode == REQUEST_CODE_GEOLOCATION) {
            NotificationCenter.getGlobalInstance().postNotificationName(granted ? NotificationCenter.locationPermissionGranted : NotificationCenter.locationPermissionDenied);
        } else if (requestCode == REQUEST_CODE_MEDIA_GEO) {
            NotificationCenter.getGlobalInstance().postNotificationName(granted ? NotificationCenter.locationPermissionGranted : NotificationCenter.locationPermissionDenied, 1);
        }
        return true;
    }

}
```
