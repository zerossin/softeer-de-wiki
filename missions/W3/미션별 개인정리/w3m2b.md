# W3M2b - Understanding of Hadoop Configuration Files

W3M2의 2번째 파트로 클러스터를 새로 만드는 게 아니라 core/hdfs/mapred/yarn-site.xml의 설정값을 이해하고 있는지 확인하는 미션이었다. 지정된 12개 설정을 바꾸는 스크립트와 바뀐 값을 검증하는 스크립트를 파이썬으로 짜는 게 핵심이었다.

브랜치는 M2a에서 팠다. 이 미션은 클러스터를 새로 만드는 게 아니라 설정 스크립트를 만드는 게 결과물이라 이미 있는 M2a 클러스터 위에 스크립트만 얹으면 됐다. 마스터 호스트명을 요구사항에 맞춰 master에서 namenode로 바꾸고 xml.etree로 설정 파일을 백업하고 수정하는 configure_hadoop.py와 hdfs getconf로 값을 검증하는 verify_hadoop.py를 작성했다.

미션 문서의 예상 출력 예시를 요구사항과 대조하다가 실제로 다른 점 두 가지를 발견했다.

- 예시가 mapreduce.job.tracker라는 속성을 검증하는데 이건 YARN 이전(Hadoop 1.x) 시절 속성이라 지금 클러스터에서는 아무 의미가 없고 요구사항에도 없는 값이었다. 실제로 요구된 mapreduce.jobhistory.address를 검증하도록 고쳤다.
- 예시가 hadoop getconf와 yarn getconf 명령을 쓰는데 직접 실행해보니 이 두 서브커맨드는 Hadoop 3.3.6에 존재하지 않았다(getconf is not COMMAND 에러). hdfs getconf만 존재하고 이게 core/hdfs/mapred/yarn 설정을 전부 합쳐서 읽어주길래 전부 hdfs로 통일했다.

작업 중 실제로 겪은 버그도 하나 있었다. dfs.namenode.name.dir 경로를 바꾸면서 새 디렉터리가 비어 있길래 그냥 재포맷했더니 클러스터 ID가 바뀌어서 이미 등록돼 있던 데이터노드들이 전부 Incompatible clusterIDs 오류로 접속을 거부당했다. 재포맷 대신 기존 네임노드 디렉터리의 내용을 새 경로로 그대로 옮기도록 고쳐서 클러스터 ID와 기존 데이터를 둘 다 보존하게 만들었다.
