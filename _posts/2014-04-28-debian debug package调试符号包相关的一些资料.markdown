```text
    




  下载LOFTER我的照片书  |

man dh_strip
https://wiki.debian.org/DebugPackage
用这个命令生成那些debian debug package的吧 

-----------------
man strip

 
   --only-keep-debug
          1.<Run "objcopy --only-keep-debug foo foo.dbg" to>
               create a file containing the debugging info.

           1.<Run "objcopy --strip-debug foo" to create a>
               stripped executable.

           1.<Run "objcopy --add-gnu-debuglink=foo.dbg foo


strip 和 objcopy 应该都可以把ELF中的debug info section给复制出来吧
------------------------
默认生成的so对应各个debug info都是放到/usr/lib/debug 目录下就可以了。gdb ，  perf这些命令会自己去这个下面去查找？？


root@debian02:/opt/inficore/map_smpp_server# dpkg-query -L libc6-dbg
/.
/usr
/usr/lib
/usr/lib/debug
/usr/lib/debug/lib
/usr/lib/debug/lib/i386-linux-gnu
/usr/lib/debug/lib/i386-linux-gnu/libutil-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libutil-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libnss_compat-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libresolv-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libnss_dns-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libm-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libnss_nis-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libnss_files-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/librt-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libpcprofile.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libSegFault.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libdl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/ld-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libcidn-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libBrokenLocale-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libthread_db-1.0.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libnsl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libnss_hesiod-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libcrypt-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libanl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libmemusage.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libpthread-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libc-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/cmov/libnss_nisplus-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libutil-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libnss_compat-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libresolv-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libnss_dns-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libm-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libnss_nis-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libnss_files-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/librt-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libpcprofile.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libSegFault.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libdl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/ld-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libcidn-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libBrokenLocale-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libthread_db-1.0.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libnsl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libnss_hesiod-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libcrypt-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libanl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libmemusage.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libpthread-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libc-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/i686/nosegneg/libnss_nisplus-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libnss_compat-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libresolv-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libnss_dns-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libm-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libnss_nis-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libnss_files-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/librt-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libpcprofile.so
/usr/lib/debug/lib/i386-linux-gnu/libSegFault.so
/usr/lib/debug/lib/i386-linux-gnu/libdl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/ld-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libcidn-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libBrokenLocale-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libthread_db-1.0.so
/usr/lib/debug/lib/i386-linux-gnu/libnsl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libnss_hesiod-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libcrypt-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libanl-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libmemusage.so
/usr/lib/debug/lib/i386-linux-gnu/libpthread-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libc-2.13.so
/usr/lib/debug/lib/i386-linux-gnu/libnss_nisplus-2.13.so
/usr/lib/debug/usr
/usr/lib/debug/usr/lib
/usr/lib/debug/usr/lib/i386-linux-gnu
/usr/lib/debug/usr/lib/i386-linux-gnu/crt1.o
/usr/lib/debug/usr/lib/i386-linux-gnu/Mcrt1.o
/usr/lib/debug/usr/lib/i386-linux-gnu/crti.o
/usr/lib/debug/usr/lib/i386-linux-gnu/Scrt1.o
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-FI-SE-A.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1161.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/HP-THAI8.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1149.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1124.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-9E.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/SJIS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM278.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM855.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/libISOIR165.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EUC-JP.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM866.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM871.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/SAMI-WS2.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1254.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO646.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM420.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM943.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/MAC-UK.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM857.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1252.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1155.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/SHIFT_JISX0213.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-5.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/INIS-CYRILLIC.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/libCNS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO-IR-209.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1371.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM9448.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1156.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM901.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM904.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-AT-DE-A.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/VISCII.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/PT154.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM922.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-7.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM424.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM280.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/UNICODE.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1144.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM865.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1047.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-AT-DE.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/UTF-7.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM4517.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM937.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1008.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CWI.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/TSCII.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CSN_369103.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1097.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EUC-JP-MS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM902.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1253.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1153.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/libGB.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-IS-FRISS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM9030.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM16804.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/HP-ROMAN8.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-DK-NO.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-8.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM284.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM4909.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM281.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM869.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EUC-CN.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM880.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM277.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1157.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM803.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1257.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/MAC-IS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1163.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/T.61.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1251.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1167.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-3.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM256.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/BIG5.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO_5427.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1388.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/BRF.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM852.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-US.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-6.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/RK1048.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ECMA-CYRILLIC.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1141.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO_5427-EXT.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GEORGIAN-ACADEMY.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/HP-TURKISH8.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1129.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/BIG5HKSCS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GEORGIAN-PS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM868.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM933.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO-2022-CN-EXT.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM864.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM275.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/INIS-8.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1390.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM875.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP10007.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP775.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM4971.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GBGBK.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1130.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1147.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM866NAV.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EUC-TW.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/UTF-32.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-ES-S.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-1.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1025.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM500.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM862.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO_10367-BOX.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM935.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1026.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM861.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1004.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM297.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM863.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM860.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM903.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-14.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM290.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/TIS-620.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/libJIS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1148.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-ES-A.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1399.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM273.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM850.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1160.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO-2022-JP.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GREEK-CCITT.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM905.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-10.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM12712.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EUC-JISX0213.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-2.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1046.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-FI-SE.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1255.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM038.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1123.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/KOI8-U.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1132.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1125.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GBK.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-15.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-9.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1146.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM5347.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM939.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IEC_P27-1.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1142.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO_5428.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1158.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1008_420.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1137.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/KOI8-R.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM870.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO-IR-197.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/NATS-DANO.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/HP-ROMAN9.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM856.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1364.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-11.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1143.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM930.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GREEK7.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO_11548-1.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-CA-FR.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM285.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/LATIN-GREEK-1.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GOST_19768-74.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-IT.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GBBIG5.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/INIS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-16.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1256.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/NATS-SEFI.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1122.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/KOI8-RU.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM921.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO-2022-CN.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GB18030.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/MAC-CENTRALEUROPE.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EUC-KR.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-UK.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/MAC-SAMI.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ASMO_449.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/JOHAB.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM423.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM9066.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/MIK.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/DEC-MCS.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/UHC.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-PT.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/UTF-16.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1140.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1250.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/libKSC.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ARMSCII-8.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP932.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM037.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO_6937-2.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-ES.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP1258.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1133.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1162.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM918.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1164.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/libJISX0213.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISIRI-3342.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO_6937.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM437.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM4899.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/KOI8-T.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/GREEK7-OLD.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/CP737.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM274.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1154.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1166.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/LATIN-GREEK.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/HP-GREEK8.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM891.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM932.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-DK-NO-A.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/EBCDIC-FR.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM874.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO-2022-KR.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1145.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM851.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO_2033.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO-2022-JP-3.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/KOI-8.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ANSI_X3.110.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/IBM1112.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-4.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/ISO8859-13.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/TCVN5712-1.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gconv/MACINTOSH.so
/usr/lib/debug/usr/lib/i386-linux-gnu/gcrt1.o
/usr/lib/debug/usr/lib/i386-linux-gnu/crtn.o
/usr/share
/usr/share/doc
/usr/share/doc/libc6-dbg
/usr/share/doc/libc6-dbg/changelog.gz
/usr/share/doc/libc6-dbg/copyright
/usr/share/doc/libc6-dbg/changelog.Debian.gz




.text代码段，数据段等应该不在这个debug info 里面的吧。
root@debian01:/opt/inficore/map_smpp_client# objdump  -s /usr/lib/debug/lib/i386-linux-gnu/libc-2.13.so | grep section
Contents of section .note.gnu.build-id:
Contents of section .note.ABI-tag:
Contents of section .comment:
Contents of section .debug_aranges:
Contents of section .debug_pubnames:
Contents of section .debug_info:
Contents of section .debug_abbrev:
Contents of section .debug_line:
Contents of section .debug_frame:
Contents of section .debug_str:
Contents of section .debug_loc:
Contents of section .debug_ranges:
 
```
